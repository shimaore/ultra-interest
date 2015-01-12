    chai = require 'chai'
    chai.use require 'chai-as-promised'
    chai.should()
    logger = require 'winston'
    pkg = require '../package.json'
    request = require 'superagent-as-promised'

    PouchDB = require 'pouchdb'
    Promise = require 'bluebird'

    seconds = 1000

    exec = (require 'exec-as-promised') logger

    do_before = ->
      wait = 3
      @timeout (wait+1)*seconds
      Promise.resolve()
      .then ->
        exec "docker run -d --name #{pkg.name}-db1 -p 127.0.0.1:5985:5984                            shimaore/couchdb"
      .then ->
        exec "docker run -d --name #{pkg.name}-db2 -p 127.0.0.1:5986:5984 --link #{pkg.name}-db1:db1 shimaore/couchdb"
      .then ->
        logger.info "Waiting #{wait} seconds for CouchDB to start."
      .delay wait*seconds
      .then ->
        request.get 'http://127.0.0.1:5985'
      .then (res) ->
        request.get 'http://127.0.0.1:5986'
      .then (res) ->
        logger.info "Both databases are ready."

    do_after = ->
      @timeout 3*seconds
      Promise.all [
        exec "docker logs #{pkg.name}-db1"
        .then ->
          exec "docker kill #{pkg.name}-db1"
        .delay 1*seconds
        .then ->
          exec "docker rm #{pkg.name}-db1"
        .delay 1*seconds
      ,
        exec "docker logs #{pkg.name}-db2"
        .then ->
          exec "docker kill #{pkg.name}-db2"
        .delay 1*seconds
        .then ->
          exec "docker rm #{pkg.name}-db2"
        .delay 1*seconds
      ]

    replicate = ->
      request
      .post 'http://127.0.0.1:5986/_replicate'
      .send
        source: 'http://db1:5984/db1'
        target: 'db2'
        continuous: true
        type: 'json'
        accept: 'json'
      .then (res) ->
        res.should.have.property 'text'
        body = JSON.parse res.text
        body.should.have.property 'ok', true
        body.should.have.property '_local_id'
        id = body._local_id
        logger.info "Replicate: #{JSON.stringify res}"

    describe 'Replication and deletion', ->
      before do_before
      after do_after
      it 'replication started before', (done) ->
        @timeout 10*seconds

        db1 = new PouchDB 'http://127.0.0.1:5985/db1'
        db2 = new PouchDB 'http://127.0.0.1:5986/db2'

        ch1 = db1.changes include_docs:true, live:true
        .on 'change', (change) ->
          logger.info "db1: #{JSON.stringify change}"
          change.should.have.property 'doc'
          change.doc.should.have.property 'data', true

        ch2 = db2.changes include_docs:true, live:true
        .on 'change', (change) ->
          logger.info "db2: #{JSON.stringify change}"
          change.doc.should.have.property 'data', true
          if change.doc._deleted
            ch1.cancel()
            ch2.cancel()
            done()

        replicate()
        .then ->
          db1.put
            _id: 'foo'
            data: true
        .then ->
          db1.get 'foo'
        .then (doc) ->
          doc.should.have.property 'data', true

          db1.get 'foo'
        .then (doc) ->
          doc._deleted = true
          db1.put doc
        .delay 5*seconds
        true

    describe.only 'Deletion, replication, deletion', ->
      before do_before
      after do_after
      it 'replication started after', (done) ->
        @timeout 10*seconds

        db1 = new PouchDB 'http://127.0.0.1:5985/db1'
        db2 = new PouchDB 'http://127.0.0.1:5986/db2'

        ch1 = db1.changes include_docs:true, live:true
        .on 'change', (change) ->
          logger.info "db1: #{JSON.stringify change}"
          change.should.have.property 'doc'
          change.doc.should.have.property 'data'

        ch2 = db2.changes include_docs:true, live:true
        .on 'change', (change) ->
          logger.info "db2: #{JSON.stringify change}"
          change.doc.should.have.property 'data'
          if change.doc._deleted and change.doc.data is 'bar2'
            ch1.cancel()
            ch2.cancel()
            done()

        db1.put
          _id: 'foo'
          t: 1
          data: 'bar1'
        .then ->
          db1.get 'foo'
        .then (doc) ->
          doc.should.have.property 'data', 'bar1'
        .then ->
          db1.get 'foo'
        .then (doc) ->
          logger.info "db1.get: #{JSON.stringify doc}"
          doc._deleted = true
          db1.put doc
        .then ->
          db1.put
            _id: 'foo'
            t: 2
            data: 'bar2'
        .then ->
          replicate()
        .then ->
          db1.get 'foo'
        .then (doc) ->
          logger.info "db1.get: #{JSON.stringify doc}"
          doc._deleted = true
          db1.put doc
        true
