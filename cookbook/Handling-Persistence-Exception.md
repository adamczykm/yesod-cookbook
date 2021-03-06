# Handling exception in persistence

Say, you have a code like this which throws an exception:

``` haskell
share [mkPersist sqlSettings, mkMigrate "migrateAll"] [persistLowerCase|
User
    age Int
    UniqueAge age
    deriving Show
|]

getUsers :: MonadIO m => ReaderT SqlBackend m [Entity User]
getUsers = selectList [] []

insertJane :: MonadIO m => ReaderT SqlBackend m ()
insertJane = insert_ $ User 40

sqliteTest :: IO ()
sqliteTest = runSqlite ":memory:" $ do
               runMigration migrateAll

               insertJane
               insertJane

               users <- getUsers
               liftIO $ print (users :: [Entity User])
```

The above code will throw an exception when executed because the uniqueness constraint is being violated:

```
$ ./sqlite-code

Migrating: CREATE TABLE "user"("id" INTEGER PRIMARY KEY,"age" INTEGER NOT NULL,CONSTRAINT "unique_age" UNIQUE ("age"))
persistent-try-bugs: SQLite3 returned ErrorConstraint while attempting to perform step.
```

You can use the [exceptions](https://hackage.haskell.org/package/exceptions) package to handle exceptions. Have the appropriate imports:

```
import Control.Exception (SomeException)
import Control.Monad.Catch
```

And the rest of the code is like this:

``` haskell
share [mkPersist sqlSettings, mkMigrate "migrateAll"] [persistLowerCase|
User
    age Int
    UniqueAge age
    deriving Show
|]

getUsers :: MonadIO m => ReaderT SqlBackend m [Entity User]
getUsers = selectList [] []

insertJane :: (MonadCatch m, MonadIO m) => ReaderT SqlBackend m ()
insertJane = (insert_ (User 40)) `catch` (\(SomeException e) -> return ())

sqliteTest :: IO ()
sqliteTest = runSqlite ":memory:" $ do
               runMigration migrateAll

               insertJane
               insertJane

               users <- getUsers
               liftIO $ print (users :: [Entity User])
```

And this time, it will execute without crashing:

``` 
$ ./sqlite-code

Migrating: CREATE TABLE "user"("id" INTEGER PRIMARY KEY,"age" INTEGER NOT NULL,CONSTRAINT "unique_age" UNIQUE ("age"))
[Entity {entityKey = UserKey {unUserKey = SqlBackendKey {unSqlBackendKey = 1}}, entityVal = User {userAge = 40}}]
```

While the above code works for `sqlite` database, it will fail when used with `PostgreSql` and others. First Let's check in the sql shell, the current data:

``` sql
perm2=# select id, age from public.user;
 id | age
----+-----
(0 rows)
```

Okay, now let's run this code:

```haskell
{-# LANGUAGE EmptyDataDecls             #-}
{-# LANGUAGE FlexibleContexts           #-}
{-# LANGUAGE GADTs                      #-}
{-# LANGUAGE GeneralizedNewtypeDeriving #-}
{-# LANGUAGE MultiParamTypeClasses      #-}
{-# LANGUAGE OverloadedStrings          #-}
{-# LANGUAGE QuasiQuotes                #-}
{-# LANGUAGE TemplateHaskell            #-}
{-# LANGUAGE TypeFamilies               #-}

module Postgresql where

import           Control.Monad.IO.Class  (liftIO)
import           Control.Monad.Logger    (runNoLoggingT)
import           Database.Persist
import           Control.Monad.Reader
import           Data.Text
import           Database.Persist.Sql
import           Database.Persist.Postgresql
import           Database.Persist.TH
import Control.Exception (SomeException)
import Control.Monad.Catch

share [mkPersist sqlSettings, mkMigrate "migrateAll"] [persistLowerCase|
User
    age Int
    UniqueAge age
    deriving Show
|]

getUsers :: MonadIO m => ReaderT SqlBackend m [Entity User]
getUsers = selectList [] []

insertJane :: (MonadCatch m, MonadIO m) => ReaderT SqlBackend m ()
insertJane = (insert_ (User 40)) `catch` (\(SomeException e) -> return ())

conn = "host=localhost dbname=perm2 user=postgres password=postgres port=5432"
       
postgreSQLTest :: IO ()
postgreSQLTest = runNoLoggingT $ withPostgresqlPool conn 10 $ liftSqlPersistMPool $ do

                   liftIO $ print "Going to insert jane twice"
                   
                   insertJane
                   insertJane -- This should cause exception 
                   
                   users <- getUsers
                   liftIO $ print users

```

Executing it:

$ ./postgresql-code

``` shellsession
"Going to insert jane twice"
persistent-try-bugs: SqlError {sqlState = "25P02", sqlExecStatus = FatalError, sqlErrorMsg = "current transaction is aborted, commands ignored until end of transaction block", sqlErrorDetail = "", sqlErrorHint = ""}
```

Note how the entire transaction is aborted. You can inspect the data in the sql shell itself:

``` sql
perm2=# select id, age from public.user;
 id | age
----+-----
(0 rows)
```

This below code shows how to handle exception in a database like `PostgreSQL`. The key is to split your queries into transactions and catching exception on the transaction.

``` haskell
{-# LANGUAGE EmptyDataDecls             #-}
{-# LANGUAGE FlexibleContexts           #-}
{-# LANGUAGE GADTs                      #-}
{-# LANGUAGE GeneralizedNewtypeDeriving #-}
{-# LANGUAGE MultiParamTypeClasses      #-}
{-# LANGUAGE OverloadedStrings          #-}
{-# LANGUAGE QuasiQuotes                #-}
{-# LANGUAGE TemplateHaskell            #-}
{-# LANGUAGE TypeFamilies               #-}

module Postgresql where

import Database.Persist
import Data.Pool (Pool)
import           Control.Monad.IO.Class  (liftIO)
import           Control.Monad.Logger    (runNoLoggingT)
import           Database.Persist
import           Control.Monad.Reader
import           Data.Text
import           Database.Persist.Sql
import           Database.Persist.Postgresql
import           Database.Persist.TH
import Control.Exception (SomeException)
import Control.Monad.Catch

share [mkPersist sqlSettings, mkMigrate "migrateAll"] [persistLowerCase|
User
    age Int
    UniqueAge age
    deriving Show
|]

getUsers :: MonadIO m => ReaderT SqlBackend m [Entity User]
getUsers = selectList [] []

insertJane :: MonadIO m => ReaderT SqlBackend m ()
insertJane = insert_ $ User 40

mapLiftSqlPersistMPool :: MonadIO m => [SqlPersistM a] -> Pool SqlBackend -> m ()
mapLiftSqlPersistMPool xs pool = mapM_ (\x -> liftSqlPersistMPool x pool) xs

conn = "host=localhost dbname=perm2 user=postgres password=postgres port=5432"
       
postgreSQLTest :: IO ()
postgreSQLTest = runNoLoggingT $ withPostgresqlPool conn 10 $ 
                 mapLiftSqlPersistMPool [transaction1, transaction2]

transaction1 :: (MonadIO m, MonadCatch m) => ReaderT SqlBackend m ()
transaction1 = (do
  liftIO $ print "Inside transaction 1"
  insertJane
  insertJane -- This should cause exception 
  liftIO $ print "This should not be printed (transaction 1)"
  u <- getUsers
  liftIO $ print u
  return ()) `catch` (\(SomeException e) -> return ())


transaction2 :: (MonadIO m, MonadCatch m) => ReaderT SqlBackend m ()
transaction2 = do
  liftIO $ print "Inside transaction 2"
  users <- getUsers
  liftIO $ print users
  liftIO $ print "This should just print fine (transaction 2)"
  return ()
```

On execution:

``` shellsession
$ ./postgresql-code
"Inside transaction 1"
"Inside transaction 2"
[]
"This should just print fine (transaction 2)"
```

