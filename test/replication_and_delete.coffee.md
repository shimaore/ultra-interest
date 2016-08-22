    chai = require 'chai'
    chai.use require 'chai-as-promised'
    chai.should()
    logger = null
    logger = require 'winston'
    pkg = require '../package.json'
    request = require 'superagent-as-promised'

    PouchDB = require 'pouchdb'
    Promise = require 'bluebird'

    port = ->
      Math.floor(Math.random()*30000+20000)
    seconds = 1000

    exec = (require 'exec-as-promised') logger

    start_servers = ([port1,port2]) ->
      wait = 3
      @timeout (wait+1)*seconds
      Promise.resolve()
      .then ->
        exec "docker run -d --name #{pkg.name}-db1-#{port1} -p 127.0.0.1:#{port1}:5984                                     shimaore/couchdb"
      .then ->
        exec "docker run -d --name #{pkg.name}-db2-#{port2} -p 127.0.0.1:#{port2}:5984 --link #{pkg.name}-db1-#{port1}:db1 shimaore/couchdb"
      .then ->
        logger?.info "Waiting #{wait} seconds for CouchDB to start."
      .delay wait*seconds
      .then ->
        request.get "http://127.0.0.1:#{port1}"
      .then (res) ->
        request.get "http://127.0.0.1:#{port2}"
      .then (res) ->
        logger?.info "Both databases are ready."

    stop_servers = ([port1,port2])->
      @timeout 3*seconds
      Promise.all ["db1-#{port1}","db2-#{port2}"].map (db) ->
        exec "docker logs #{pkg.name}-#{db}"
        .catch -> true
        .then ->
          exec "docker kill #{pkg.name}-#{db}"
        .catch -> true
        .delay 1*seconds
        .then ->
          exec "docker rm #{pkg.name}-#{db}"
        .catch -> true
        .delay 1*seconds

    replicate = (port2) ->
      request
      .post "http://127.0.0.1:#{port2}/_replicate"
      .send
        source: 'http://db1:5984/db1'
        target: 'db2'
        continuous: true
      .accept 'json'
      .then (res) ->
        res.should.have.property 'text'
        body = JSON.parse res.text
        body.should.have.property 'ok', true
        body.should.have.property '_local_id'
        id = body._local_id
        logger?.info "Replicate: #{JSON.stringify res}"

        cancel: ->
          request
          .post "http://127.0.0.1:#{port2}/_replicate"
          .send
            source: 'http://db1:5984/db1'
            target: 'db2'
            continuous: true
            cancel: true

This first test does early replication, then two add/delete steps
=================================================================

It should always work.

    describe 'Replication (1)', ->

      ports = [port1 = port(), port2 = port()]
      before ->
        start_servers.call this, ports
      after ->
        stop_servers.call this, ports

      it 'should show everything when started before', (done) ->
        @timeout 10*seconds

        db1 = new PouchDB "http://127.0.0.1:#{port1}/db1"
        db2 = new PouchDB "http://127.0.0.1:#{port2}/db2"

        seen1 = {}

        ch1 = db1.changes include_docs:true, live:true
        .on 'change', (change) ->
          logger?.info "db1: #{JSON.stringify change}"
          change.should.have.property 'doc'
          change.doc.should.have.property 'data'
          seen1[change.doc.data] = true if change.doc._deleted

        seen2 = {}

        ch2 = db2.changes include_docs:true, live:true
        .on 'change', (change) ->
          logger?.info "db2: #{JSON.stringify change}"
          change.should.have.property 'doc'
          change.doc.should.have.property 'data'
          seen2[change.doc.data] = true if change.doc._deleted

          if change.doc._deleted and change.doc.data is 'bar2'
            Promise.delay 500
            .then ->
              ch1.cancel()
              ch2.cancel()
              rep.cancel()
              seen1.should.have.property 'bar1', true
              seen1.should.have.property 'bar2', true
              seen2.should.have.property 'bar1', true
              seen2.should.have.property 'bar2', true
              done()
            .catch done

Replicate

        rep = replicate port2

Value: data:"bar1"

        .then ->
          db1.put
            _id: 'foo'
            data: 'bar1'
        .then ->
          db1.get 'foo'
        .then (doc) ->
          doc.should.have.property 'data', 'bar1'
          doc._deleted = true
          db1.put doc

New value: data:"bar2".

        .then ->
          db1.put
            _id: 'foo'
            data: 'bar2'
        .then ->
          db1.get 'foo'
        .then (doc) ->
          doc.should.have.property 'data', 'bar2'
          doc._deleted = true
          db1.put doc
        .delay 5*seconds

This second test does a first add/delete round, starts replication, then does a second add/delete round.
========================================================================================================

    describe 'Replication (2)', ->

      ports = [port1 = port(), port2 = port()]
      before ->
        start_servers.call this, ports
      after ->
        stop_servers.call this, ports

      it 'should everything when started in the middle', (done) ->
        @timeout 10*seconds

        db1 = new PouchDB "http://127.0.0.1:#{port1}/db1"
        db2 = new PouchDB "http://127.0.0.1:#{port2}/db2"

        seen1 = {}

        ch1 = db1.changes include_docs:true, live:true
        .on 'change', (change) ->
          logger?.info "db1: #{JSON.stringify change}"
          change.should.have.property 'doc'
          change.doc.should.have.property 'data'
          seen1[change.doc.data] = true if change.doc._deleted

        seen2 = {}

        ch2 = db2.changes include_docs:true, live:true
        .on 'change', (change) ->
          logger?.info "db2: #{JSON.stringify change}"
          change.should.have.property 'doc'
          change.doc.should.have.property 'data'
          seen2[change.doc.data] = true if change.doc._deleted

          if change.doc._deleted and change.doc.data is 'bar2'
            Promise.delay 500
            .then ->
              ch1.cancel()
              ch2.cancel()
              rep.cancel()
              seen1.should.have.property 'bar1', true
              seen1.should.have.property 'bar2', true
              seen2.should.have.property 'bar1', true
              seen2.should.have.property 'bar2', true
              done()
            .catch done

        rep = null

        Promise.resolve()

Value: data:"bar1"

        .then ->
          db1.put
            _id: 'foo'
            data: 'bar1'
        .then ->
          db1.get 'foo'
        .then (doc) ->
          doc.should.have.property 'data', 'bar1'
          doc._deleted = true
          db1.put doc

Replicate

        .then ->
          rep = replicate port2
        .delay 2*seconds

New value: data:"bar2".

        .then ->
          db1.put
            _id: 'foo'
            data: 'bar2'
        .then ->
          db1.get 'foo'
        .then (doc) ->
          doc.should.have.property 'data', 'bar2'
          doc._deleted = true
          db1.put doc
        .delay 5*seconds

This test does multiple adds in the first round, starts replication, then does a second add/delete round.
========================================================================================================

    describe.only 'Replication (M)', ->

      ports = [port1 = port(), port2 = port()]
      before ->
        start_servers.call this, ports
      after ->
        stop_servers.call this, ports

      it 'should everything when started in the middle', (done) ->
        @timeout 18*seconds

        db1 = new PouchDB "http://127.0.0.1:#{port1}/db1"
        db2 = new PouchDB "http://127.0.0.1:#{port2}/db2"

        seen1 = {}

        ch1 = db1.changes include_docs:true, live:true
        .on 'change', (change) ->
          logger?.info "db1: #{JSON.stringify change}"
          change.should.have.property 'doc'
          change.doc.should.have.property 'data'
          seen1[change.doc.data] = true if change.doc._deleted

        seen2 = {}

        ch2 = db2.changes include_docs:true, live:true
        .on 'change', (change) ->
          logger?.info "db2: #{JSON.stringify change}"
          change.should.have.property 'doc'
          change.doc.should.have.property 'data'
          seen2[change.doc.data] = true if change.doc._deleted

        rep = null

        it = Promise.resolve()

Value: data:"bar1"

          .then ->
            db1.put
              _id: 'foo'
              data: 'bar1'

        for i in [1..34]
          it = it.then ->
              db1.get 'foo'
            .then (doc) ->
              doc.should.have.property 'data', 'bar1'
              db1.put doc
            .delay 50

        it = it.then ->
            db1.get 'foo'
          .then (doc) ->
            doc.should.have.property 'data', 'bar1'
            doc._deleted = true
            db1.put doc
          .delay 100

Value: data:"bar2"

          .then ->
            db1.put
              _id: 'foo'
              data: 'bar2'

        for i in [1..59]
          it = it.then ->
              db1.get 'foo'
            .then (doc) ->
              doc.should.have.property 'data', 'bar2'
              db1.put doc
            .delay 50

        it = it.then ->
            db1.get 'foo'
          .then (doc) ->
            doc.should.have.property 'data', 'bar2'
            doc._deleted = true
            db1.put doc
          .delay 100

Value: data:"bar3"

          .then ->
            db1.put
              _id: 'foo'
              data: 'bar3'

        for i in [1..23]
          it = it.then ->
              db1.get 'foo'
            .then (doc) ->
              doc.should.have.property 'data', 'bar3'
              db1.put doc
            .delay 50

        it = it.then ->
            db1.get 'foo'
          .then (doc) ->
            doc.should.have.property 'data', 'bar3'
            doc._deleted = true
            db1.put doc
          .delay 100

Replicate

        it = it.then ->
          rep = replicate port2
        .delay 2*seconds

New value: data:"bar2".

        .then ->
          db1.put
            _id: 'foo'
            data: 'barL'
        .delay 100
        .then ->
          db1.get 'foo'
        .then (doc) ->
          doc.should.have.property 'data', 'barL'
          doc._deleted = true
          db1.put doc
        .delay 5*seconds
        .then ->
          ch1.cancel()
          ch2.cancel()
          rep.cancel()
          seen1.should.have.property 'bar1', true
          seen1.should.have.property 'bar2', true
          seen1.should.not.have.property 'bar3'
          seen1.should.not.have.property 'barL'
          seen2.should.not.have.property 'bar1'
          seen2.should.have.property 'bar2', true
          seen2.should.not.have.property 'bar3'
          seen2.should.not.have.property 'barL'
          done()
        .catch done


This test does the first add/delete round, then a second add, then starts replication, then the second delete
=============================================================================================================

    describe 'Replication (3)', ->
      ports = [port1 = port(), port2 = port()]
      before ->
        start_servers.call this, ports
      after ->
        stop_servers.call this, ports

      it 'should show everything when started after second insert', (done) ->
        @timeout 10*seconds

        db1 = new PouchDB "http://127.0.0.1:#{port1}/db1"
        db2 = new PouchDB "http://127.0.0.1:#{port2}/db2"

        seen1 = {}

        ch1 = db1.changes include_docs:true, live:true
        .on 'change', (change) ->
          logger?.info "db1: #{JSON.stringify change}"
          change.should.have.property 'doc'
          change.doc.should.have.property 'data'
          seen1[change.doc.data] = true if change.doc._deleted

        seen2 = {}

        ch2 = db2.changes include_docs:true, live:true
        .on 'change', (change) ->
          logger?.info "db2: #{JSON.stringify change}"
          change.should.have.property 'doc'
          change.doc.should.have.property 'data'
          seen2[change.doc.data] = true if change.doc._deleted

          if change.doc._deleted and change.doc.data is 'bar2'
            Promise.delay 500
            .then ->
              ch1.cancel()
              ch2.cancel()
              rep.cancel()
              seen1.should.have.property 'bar1', true
              seen1.should.have.property 'bar2', true
              seen2.should.not.have.property 'bar1'
              seen2.should.have.property 'bar2', true
              done()
            .catch done

        rep = null

        Promise.resolve()

First round: data:bar1

        .then ->
          db1.put
            _id: 'foo'
            data: 'bar1'
        .then ->
          db1.get 'foo'
        .then (doc) ->
          logger?.info "db1.get: #{JSON.stringify doc}"
          doc.should.have.property 'data', 'bar1'
          doc._deleted = true
          db1.put doc

Second round: only add

        .then ->
          db1.put
            _id: 'foo'
            data: 'bar2'

Start replication.

        .then ->
          rep = replicate port2

Second round: only delete.

        .then ->
          db1.get 'foo'
        .then (doc) ->
          logger?.info "db1.get: #{JSON.stringify doc}"
          doc.should.have.property 'data', 'bar2'
          doc._deleted = true
          db1.put doc

In this one we re-use the last database and use a filtered replication.

      it.skip 'filtered replication', (done) ->
        @timeout 10*seconds

        db3 = new PouchDB "http://127.0.0.1:#{port2}/db3"

        ch3 = db3.changes include_docs:true, live:true
        .on 'change', (change) ->
          logger?.info "db2: #{JSON.stringify change}"
          change.doc.should.have.property 'data'
          if change.doc._deleted and change.doc.data is 'bar2'
            ch3.cancel()
            done()

        request
        .post "http://127.0.0.1:#{port2}/_replicate"
        .send
          source: "http://db1:#{port1}/db1"
          target: 'db3'
          continuous: true
        .accept 'json'
        .then (res) ->
          res.should.have.property 'text'
          body = JSON.parse res.text
          body.should.have.property 'ok', true
          body.should.have.property '_local_id'
          id = body._local_id
          logger?.info "Replicate: #{JSON.stringify res}"
