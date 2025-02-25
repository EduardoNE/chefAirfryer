# SQLite plugin in Markdown (litcoffee)

#### Use coffee compiler to compile this directly into Javascript

#### License for common script: MIT or Apache

# Top-level SQLite plugin objects

## root window object:

    root = @

## constant(s):

    READ_ONLY_REGEX = /^\s*(?:drop|delete|insert|update|create)\s/i

    # per-db state
    DB_STATE_INIT = "INIT"
    DB_STATE_OPEN = "OPEN"

## global(s):

    # per-db map of locking and queueing
    # XXX NOTE: This is NOT cleaned up when a db is closed and/or deleted.
    # If the record is simply removed when a db is closed or deleted,
    # it will cause some test failures and may break large-scale
    # applications that repeatedly open and close the database.
    # [BUG #210] TODO: better to abort and clean up the pending transaction state.
    # XXX TBD this will be renamed and include some more per-db state.
    txLocks = {}

## utility functions:

    # Errors returned to callbacks must conform to `SqlError` with a code and message.
    # Some errors are of type `Error` or `string` and must be converted.
    newSQLError = (error, code) ->
      sqlError = error
      code = 0 if !code # unknown by default

      if !sqlError
        sqlError = new Error "a plugin had an error but provided no response"
        sqlError.code = code

      if typeof sqlError is "string"
        sqlError = new Error error
        sqlError.code = code

      if !sqlError.code && sqlError.message
        sqlError.code = code

      if !sqlError.code && !sqlError.message
        sqlError = new Error "an unknown error was returned: " + JSON.stringify(sqlError)
        sqlError.code = code

      return sqlError

    nextTick = window.setImmediate || (fun) ->
      window.setTimeout(fun, 0)
      return

    ###
      Utility that avoids leaking the arguments object. See
      https://www.npmjs.org/package/argsarray
    ###
    argsArray = (fun) ->
      return ->
        len = arguments.length
        if len
          args = []
          i = -1
          while ++i < len
            args[i] = arguments[i]
          return fun.call this, args
        else
          return fun.call this, []

## SQLite plugin db-connection handle

#### NOTE: there can be multipe SQLitePlugin db-connection handles per open db.

#### SQLite plugin db connection handle object is defined by a constructor function and prototype member functions:

    SQLitePlugin = (openargs, openSuccess, openError) ->
      # console.log "SQLitePlugin openargs: #{JSON.stringify openargs}"

      if !(openargs and openargs['name'])
        throw newSQLError "Cannot create a SQLitePlugin db instance without a db name"

      dbname = openargs.name

      @openargs = openargs
      @dbname = dbname

      @openSuccess = openSuccess
      @openError = openError

      @openSuccess or
        @openSuccess = ->
          console.log "DB opened: " + dbname
          return

      @openError or
        @openError = (e) ->
          console.log e.message
          return

      @open @openSuccess, @openError
      return

    SQLitePlugin::databaseFeatures = isSQLitePluginDatabase: true

    # Keep track of state of open db connections
    # XXX TBD this will be moved and renamed or
    # combined with txLocks.
    SQLitePlugin::openDBs = {}

    SQLitePlugin::addTransaction = (t) ->
      if !txLocks[@dbname]
        txLocks[@dbname] = {
          queue: []
          inProgress: false
        }
      txLocks[@dbname].queue.push t
      if @dbname of @openDBs && @openDBs[@dbname] isnt DB_STATE_INIT
        # XXX TODO: only when queue has length of 1 [and test it!!]
        @startNextTransaction()

      else
        if @dbname of @openDBs
          console.log 'new transaction is waiting for open operation'
        else
          # XXX TBD TODO: in this case (which should not happen), should abort and discard the transaction.
          console.log 'database is closed, new transaction is [stuck] waiting until db is opened again!'
      return

    SQLitePlugin::transaction = (fn, error, success) ->
      if !@openDBs[@dbname]
        error newSQLError 'database not open'
        return

      @addTransaction new SQLitePluginTransaction(this, fn, error, success, true, false)
      return

    SQLitePlugin::readTransaction = (fn, error, success) ->
      if !@openDBs[@dbname]
        error newSQLError 'database not open'
        return

      @addTransaction new SQLitePluginTransaction(this, fn, error, success, true, true)
      return

    SQLitePlugin::startNextTransaction = ->
      self = @

      nextTick =>
        if !(@dbname of @openDBs) || @openDBs[@dbname] isnt DB_STATE_OPEN
          console.log 'cannot start next transaction: database not open'
          return

        txLock = txLocks[self.dbname]
        if !txLock
          console.log 'cannot start next transaction: database connection is lost'
          # XXX TBD TODO (BUG #210/??): abort all pending transactions with error cb [and test!!]
          # @abortAllPendingTransactions()
          return

        else if txLock.queue.length > 0 && !txLock.inProgress
          # start next transaction in q
          txLock.inProgress = true
          txLock.queue.shift().start()
        return

      return

    SQLitePlugin::abortAllPendingTransactions = ->
      # extra debug info:
      # if txLocks[@dbname] then console.log 'abortAllPendingTransactions with transaction queue length: ' + txLocks[@dbname].queue.length
      # else console.log 'abortAllPendingTransactions with no transaction lock state'

      txLock = txLocks[@dbname]
      if !!txLock && txLock.queue.length > 0
        # XXX TODO: what to do in case there is a (stray) transaction in progress?
        #console.log 'abortAllPendingTransactions - cleanup old transaction(s)'
        for tx in txLock.queue
          tx.abortFromQ newSQLError 'Invalid database handle'

        # XXX TODO: consider cleaning up (delete) txLocks[@dbname] resource,
        # in case it is known there are no more pending transactions
        txLock.queue = []
        txLock.inProgress = false

      return

    SQLitePlugin::open = (success, error) ->
      if @dbname of @openDBs
        console.log 'database already open: ' + @dbname

        # for a re-open run the success cb async so that the openDatabase return value
        # can be used in the success handler as an alternative to the handler's
        # db argument
        nextTick =>
          success @
          return

      else
        console.log 'OPEN database: ' + @dbname

        opensuccesscb = =>
          # NOTE: the db state is NOT stored (in @openDBs) if the db was closed or deleted.
          # console.log 'OPEN database: ' + @dbname + ' succeeded'

          #if !@openDBs[@dbname] then call open error cb, and abort pending tx if any
          if !@openDBs[@dbname]
            console.log 'database was closed during open operation'
            # XXX TODO [BUG #210] (and test!!):
            # if !!error then error newSQLError 'database closed during open operation'
            # @abortAllPendingTransactions()

          if @dbname of @openDBs
            @openDBs[@dbname] = DB_STATE_OPEN

          if !!success then success @

          txLock = txLocks[@dbname]
          if !!txLock && txLock.queue.length > 0 && !txLock.inProgress
            @startNextTransaction()
          return

        openerrorcb = =>
          console.log 'OPEN database: ' + @dbname + ' failed, aborting any pending transactions'
          # XXX TODO: newSQLError missing the message part!
          if !!error then error newSQLError 'Could not open database'
          delete @openDBs[@dbname]
          @abortAllPendingTransactions()
          return

        # store initial DB state:
        @openDBs[@dbname] = DB_STATE_INIT

        cordova.exec opensuccesscb, openerrorcb, "SQLitePlugin", "open", [ @openargs ]

      return

    SQLitePlugin::close = (success, error) ->
      if @dbname of @openDBs
        if txLocks[@dbname] && txLocks[@dbname].inProgress
          # XXX TBD: wait for current tx then close (??)
          console.log 'cannot close: transaction is in progress'
          error newSQLError 'database cannot be closed while a transaction is in progress'
          return

        console.log 'CLOSE database: ' + @dbname

        # XXX [BUG #209] closing one db handle disables other handles to same db
        delete @openDBs[@dbname]

        if txLocks[@dbname] then console.log 'closing db with transaction queue length: ' + txLocks[@dbname].queue.length
        else console.log 'closing db with no transaction lock state'

        # XXX [BUG #210] TODO: when closing or deleting a db, abort any pending transactions [and test it!!]

        cordova.exec success, error, "SQLitePlugin", "close", [ { path: @dbname } ]

      else
        console.log 'cannot close: database is not open'
        if error then nextTick -> error()

      return

    SQLitePlugin::executeSql = (statement, params, success, error) ->
      # XXX TODO: better to capture the result, and report it once
      # the transaction has completely finished.
      # This would fix BUG #204 (cannot close db in db.executeSql() callback).
      mysuccess = (t, r) -> if !!success then success r
      myerror = (t, e) -> if !!error then error e

      myfn = (tx) ->
        tx.addStatement(statement, params, mysuccess, myerror)
        return

      @addTransaction new SQLitePluginTransaction(this, myfn, null, null, false, false)
      return

## SQLite plugin transaction object for batching:

    SQLitePluginTransaction = (db, fn, error, success, txlock, readOnly) ->
      if typeof(fn) != "function"
        ###
        This is consistent with the implementation in Chrome -- it
        throws if you pass anything other than a function. This also
        prevents us from stalling our txQueue if somebody passes a
        false value for fn.
        ###
        throw newSQLError "transaction expected a function"

      @db = db
      @fn = fn
      @error = error
      @success = success
      @txlock = txlock
      @readOnly = readOnly
      @executes = []

      if txlock
        @addStatement "BEGIN", [], null, (tx, err) ->
          throw newSQLError "unable to begin transaction: " + err.message, err.code

      return

    SQLitePluginTransaction::start = ->
      try
        @fn this
        @run()
      catch err
        # If "fn" throws, we must report the whole transaction as failed.
        txLocks[@db.dbname].inProgress = false
        @db.startNextTransaction()
        if @error
          @error newSQLError err
      return

    SQLitePluginTransaction::executeSql = (sql, values, success, error) ->

      if @finalized
        throw {message: 'InvalidStateError: DOM Exception 11: This transaction is already finalized. Transactions are committed after its success or failure handlers are called. If you are using a Promise to handle callbacks, be aware that implementations following the A+ standard adhere to run-to-completion semantics and so Promise resolution occurs on a subsequent tick and therefore after the transaction commits.', code: 11}
        return

      if @readOnly && READ_ONLY_REGEX.test(sql)
        @handleStatementFailure(error, {message: 'invalid sql for a read-only transaction'})
        return

      @addStatement(sql, values, success, error)
      return

    # This method adds the SQL statement to the transaction queue but does not check for
    # finalization since it is used to execute COMMIT and ROLLBACK.
    SQLitePluginTransaction::addStatement = (sql, values, success, error) ->

      qid = @executes.length

      params = []
      if !!values && values.constructor == Array
        for v in values
          t = typeof v
          params.push (
            if v == null || v == undefined || t == 'number' || t == 'string' then v
            else if v instanceof Blob then v.valueOf()
            else v.toString()
          )

      @executes.push
        success: success
        error: error
        qid: qid

        sql: sql
        params: params

      return

    SQLitePluginTransaction::handleStatementSuccess = (handler, response) ->
      if !handler
        return

      rows = response.rows || []
      payload =
        rows:
          item: (i) ->
            rows[i]

          length: rows.length

        rowsAffected: response.rowsAffected or 0
        insertId: response.insertId or undefined

      handler this, payload

      return

    SQLitePluginTransaction::handleStatementFailure = (handler, response) ->
      if !handler
        throw newSQLError "a statement with no error handler failed: " + response.message, response.code
      if handler(this, response) isnt false
        throw newSQLError "a statement error callback did not return false: " + response.message, response.code
      return

    SQLitePluginTransaction::run = ->
      txFailure = null

      tropts = []
      batchExecutes = @executes
      waiting = batchExecutes.length
      @executes = []
      tx = this

      handlerFor = (index, didSucceed) ->
        (response) ->
          try
            if didSucceed
              tx.handleStatementSuccess batchExecutes[index].success, response
            else
              tx.handleStatementFailure batchExecutes[index].error, newSQLError(response)
          catch err
            if !txFailure
              txFailure = newSQLError(err)

          if --waiting == 0
            if txFailure
              tx.abort txFailure
            else if tx.executes.length > 0
              # new requests have been issued by the callback
              # handlers, so run another batch.
              tx.run()
            else
              tx.finish()

          return

      i = 0

      mycbmap = {}

      while i < batchExecutes.length
        request = batchExecutes[i]

        qid = request.qid

        mycbmap[qid] =
          success: handlerFor(i, true)
          error: handlerFor(i, false)

        tropts.push
          qid: qid
          sql: request.sql
          params: request.params

        i++

      mycb = (result) ->
        #console.log "mycb result #{JSON.stringify result}"

        for r in result
          type = r.type
          qid = r.qid
          res = r.result

          q = mycbmap[qid]

          if q
            if q[type]
              q[type] res

        return

      cordova.exec mycb, null, "SQLitePlugin", "backgroundExecuteSqlBatch", [{dbargs: {dbname: @db.dbname}, executes: tropts}]

      return

    SQLitePluginTransaction::abort = (txFailure) ->
      if @finalized then return
      tx = @

      succeeded = (tx) ->
        txLocks[tx.db.dbname].inProgress = false
        tx.db.startNextTransaction()
        if tx.error then tx.error txFailure
        return

      failed = (tx, err) ->
        txLocks[tx.db.dbname].inProgress = false
        tx.db.startNextTransaction()
        if tx.error then tx.error newSQLError("error while trying to roll back: " + err.message, err.code)
        return

      @finalized = true

      if @txlock
        @addStatement "ROLLBACK", [], succeeded, failed
        @run()
      else
        succeeded(tx)

      return

    SQLitePluginTransaction::finish = ->
      if @finalized then return
      tx = @

      succeeded = (tx) ->
        txLocks[tx.db.dbname].inProgress = false
        tx.db.startNextTransaction()
        if tx.success then tx.success()
        return

      failed = (tx, err) ->
        txLocks[tx.db.dbname].inProgress = false
        tx.db.startNextTransaction()
        if tx.error then tx.error newSQLError("error while trying to commit: " + err.message, err.code)
        return

      @finalized = true

      if @txlock
        @addStatement "COMMIT", [], succeeded, failed
        @run()
      else
        succeeded(tx)

      return

    SQLitePluginTransaction::abortFromQ = (sqlerror) ->
      # NOTE: since the transaction is waiting in the queue,
      # the transaction function containing the SQL statements
      # would not be run yet. Simply report the transaction error.
      if @error
        @error sqlerror

      return

## SQLite plugin object factory:

    dblocations = [ "docs", "libs", "nosync" ]

    SQLiteFactory =
      ###
      NOTE: this function should NOT be translated from Javascript
      back to CoffeeScript by js2coffee.
      If this function is edited in Javascript then someone will
      have to translate it back to CoffeeScript by hand.
      ###
      opendb: argsArray (args) ->
        if args.length < 1 then return null

        first = args[0]
        openargs = null
        okcb = null
        errorcb = null

        if first.constructor == String
          openargs = {name: first}

          if args.length >= 5
            okcb = args[4]
            if args.length > 5 then errorcb = args[5]

        else
          openargs = first

          if args.length >= 2
            okcb = args[1]
            if args.length > 2 then errorcb = args[2]

        dblocation = if !!openargs.location then dblocations[openargs.location] else null
        openargs.dblocation = dblocation || dblocations[0]

        if !!openargs.createFromLocation and openargs.createFromLocation == 1
          openargs.createFromResource = "1"

        if !!openargs.androidDatabaseImplementation and openargs.androidDatabaseImplementation == 2
          openargs.androidOldDatabaseImplementation = 1

        if !!openargs.androidLockWorkaround and openargs.androidLockWorkaround == 1
          openargs.androidBugWorkaround = 1

        new SQLitePlugin openargs, okcb, errorcb

      deleteDb: (first, success, error) ->
        args = {}

        if first.constructor == String
          #console.log "delete db name: #{first}"
          args.path = first
          args.dblocation = dblocations[0]

        else
          #console.log "delete db args: #{JSON.stringify first}"
          if !(first and first['name']) then throw new Error "Please specify db name"
          args.path = first.name
          dblocation = if !!first.location then dblocations[first.location] else null
          args.dblocation = dblocation || dblocations[0]

        # XXX [BUG #210] TODO: when closing or deleting a db, abort any pending transactions (with error callback)
        delete SQLitePlugin::openDBs[args.path]
        cordova.exec success, error, "SQLitePlugin", "delete", [ args ]

## Exported API:

    root.sqlitePlugin =
      sqliteFeatures:
        isSQLitePlugin: true

      openDatabase: SQLiteFactory.opendb
      deleteDatabase: SQLiteFactory.deleteDb

## vim directives

#### vim: set filetype=coffee :
#### vim: set expandtab :

