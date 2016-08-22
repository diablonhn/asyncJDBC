asyncJDBC
=========
asyncJDBC is a fast asynchronous JDBC connection pool with completely fair
scheduling for reactive applications.  It is built with
`Baratine <http://baratine.io/>`_, a reactive service platform.

Features
--------

* completely fair scheduling with FIFO (first-in, first-out)
* supra-consistent response times and behavior with no long tail distribution
* non-blocking design
* callback-based and synchronous interfaces
* familiar JDBC programming

asyncJDBC's performance is consistent across all levels of concurrency by
design.  Under high concurrency, you need completely fair scheduling.
asyncJDBC accomplishes this by pushing requests onto a queue and processing
them in order.

The fastest and most feature-rich synchronous JDBC connection pool out there is
`HikariCP <https://github.com/brettwooldridge/HikariCP>`_, by far.  In terms of
raw performance, it is much faster than any other connection pool out there,
including asyncJDBC.

.. image:: https://github.com/diablonhn/asyncJDBC/wiki/asyncjdbc-vs-hikaricp-vs-dbcp.png

But absolute raw performance doesn't always give the complete picture.  The request time
distribution is more important for reliable, service-level-agreement (SLA) performance.

.. image:: https://github.com/diablonhn/asyncJDBC/wiki/asyncjdbc-vs-hikaricp-distribution.png

Digging deeper into the numbers, HikariCP has excellent raw performance.  However, it has a
long tail where requests take up to 700ms.  In some runs, it went up to as high as 8 seconds!
This can lead to sporadic timeouts due to no fault of the application.  The benchmark uses a
low concurrency of only 100 clients and pool size of 16, so the problem only gets worser with
higher load.  asyncJDBC, on the other hand, has no long tail whatsoever because **100%** of
the requests complete in under 35ms.

Completely Fair Scheduling
--------------------------
asyncJDBC's queue-based scheduling ensures that it processes requests in order.  This
gives a hard upper bound on the time for a query to start executing.  Given:

* T\ :subscript:`query`  : maximum execution time of a query
* Q\ :subscript:`depth`  : queue depth (number of concurrent clients)
* N\ :subscript:`pool`   : number of workers in the queue

In isolation, the maximum queue time is:
  
    T\ :subscript:`max` = T\ :subscript:`query` * (Q\ :subscript:`depth` / N\ :subscript:`pool`)

Plugging in some numbers:

* T\ :subscript:`query`   =   10ms
* Q\ :subscript:`depth`   =   1000
* N\ :subscript:`pool`    =   100

Tmax would be 10ms * (1000 / 100) = **100 ms**, which is the absolute worst case
scenario.  Having an upper bound is every operations engineers' wish come true.

Usage
-----
asyncJDBC is intuitive to Java developers because it basically wraps JDBC around an
async interface.  However, asyncJDBC comes with both async and sync methods:

.. code-block:: java

    void execute(Result<Integer> result, String sql, Object ... params);
    int execute(String sql, Object ... params);
  
    void query(Result<ResultSet> result, String sql, Object ... params);
    ResultSet query(String sql, Object ... params);
  
    <T> void query(Result<T> result, SqlFunction<T> fun);
    <T> T query(SqlFunction<T> fun);
  
    void stats(Result<JdbcStat> result);
    JdbcStat stats();

The async methods accept a ``Result`` callback and return ``void``.

Simple Async Query
------------------
A simple query returns an offline ``java.sql.ResultSet`` that is fully read
into memory.  To send a simple query:

.. code-block:: java

    public void main(String[] args) throws Exception {
      JdbcServiceSync jdbc = AsyncJdbc.builder().url(url).poolSize(32).create();
    
      jdbc.query(this::onResult, "SELECT * from testTable");
    }
  
    private void onResult(ResultSet rs, Throwable fail) {
      if (fail != null) {
        // do something
      }
      else {
        try {
          while (rs.next()) {
            System.out.println(rs.getString(1));
          }
        }
        catch (SQLException e) {
          e.printStackTrace();
        }
      }
    }

The above code uses a method reference as the callback; a JDK8 lambda would
work just as well.

Working With the Connection Directly
------------------------------------

.. code-block:: java

    public void main(String[] args) throws Exception {
      JdbcServiceSync jdbc = AsyncJdbc.builder().url(url).poolSize(32).create();
    
      jdbc.query(this::onResult, this::sqlFunction);
    }
  
    private String sqlFunction(Connection conn) throws Exception {
      PreparedStatement stmt = conn.prepareStatement("SELECT * FROM testTable");
      
      stmt.execute();
      
      ResultSet rs = stmt.getResultSet();
      
      return rs.next().getString(1);
    }
  
    private String void onResult(String value, Throwable fail) {
      if (fail != null) {
        // do something
      }
      else {
        System.out.println(value);
      }
    }

Fire-and-Forget Query
---------------------

To run a query asynchronously without caring for the result, just pass in a ``Result.ignore()`` callback::

    jdbc.query(Result.ignore(), "SELECT * FROM testTable");

Simple Sync Query
-----------------

.. code-block:: java

    public void main(String[] args) throws Exception {
      JdbcServiceSync jdbc = AsyncJdbc.builder().url(url).poolSize(32).create();
    
      ResultSet rs = jdbc.query("SELECT * from testTable");
    }

Benchmark Parameters
--------------------

The benchmark uses the following parameters:

* 1000 concurrent clients (blocking for HikariCP, async for asyncJDBC)
* acquire connection
* open statement
* close statement
* release connection

The goal of the benchmark is to see how the connection pool behaves under heavy resource
contention.

Roadmap
-------

DataSource interface for use in Resin, Tomcat, Spring, Play Framework, etc. (if there is demand)???

Support
-------
For discussions, feature requests, bug reports, or just to say that you find this library useful, please open a new issue in GitHub
`Issues <https://github.com/diablonhn/asyncJDBC/issues>`_.
