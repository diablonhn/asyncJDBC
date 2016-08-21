asyncJDBC
=========
asyncJDBC is a fast asynchronous JDBC connection pool for reactive applications.
It is built with `Baratine <http://baratine.io/>`_, a reactive service platform.

Features
--------

* completely fair scheduling with FIFO (first-in, first-out)
* supra-consistent response times and behavior
* non-blocking design
* callback-based and synchronous interfaces
* familiar JDBC programming

asyncJDBC's performance is consistent across all levels of concurrency by
design.  Under high concurrency, you need completely fair scheduling.
asyncJDBC accomplishes this by pushing requests onto a queue and processing
them in order.

The fastest and most feature-rich synchronous JDBC connection pool out there is
`HikariCP <https://github.com/brettwooldridge/HikariCP>`_, by far.  It beats
the pants off of everyone else, including asyncJDBC.  However under high
concurrency, asyncJDBC is faster than HikariCP. More importantly, asyncJDBC
exhibits more consistent response times, which is extremely valuable in a
production environment.

.. image:: https://github.com/diablonhn/asyncJDBC/wiki/asyncjdbc-vs-hikaricp.png

The benchmark for the graph above uses the following parameters:

* 1000 concurrent clients (blocking for HikariCP, async for asyncJDBC)
* acquire connection
* open statement
* close statement
* release connection

The goal of the benchmark is to see how the connection pool behaves under heavy resource
contention.

Completely Fair Scheduling
--------------------------
asyncJDBC's queue-based scheduling ensures that it processes requests in order.  This
gives a hard upper bound on the time for a query to start executing.  Given:

* Tquery  : maximum execution time of a query
* Qdepth  : queue depth (number of concurrent clients)
* Npool   : number of workers in the queue

The maximum queue time is: Tmax = Tquery * (Qdepth / Npool).  For example:

* Tquery   =   100ms
* Qdepth   =   1000
* Npool    =   100

Tmax would 100ms * (1000 / 100) = *1 second*, which is the absolute worst case
scenario.  Having an upper bound is every operation engineers' wish come true.

Usage
-----
asyncJDBC is easy to use because it basically wraps JDBC around an async
interface.  asyncJDBC comes with both async and sync methods:

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

Simple Sync Query
-----------------

.. code-block:: java

    public void main(String[] args) throws Exception {
      JdbcServiceSync jdbc = AsyncJdbc.builder().url(url).poolSize(32).create();
    
      ResultSet rs = jdbc.query("SELECT * from testTable");
    }

Support
-------
For discussions or bug reports, please open a new issue in GitHub `Issues <https://github.com/diablonhn/asyncJDBC/issues>`_.
