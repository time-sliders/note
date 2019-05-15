# 用法

```java
public void testJDBCTrans() {
    try {
        /*
         * 1. Obtaining a connection from the DriverManager
         */
        Connection conn =
                DriverManager.getConnection("jdbc:mysql://localhost:3306/zhw?" +
                        "user=root&password=root");

        conn.setAutoCommit(false);// open transactional

        /*
         * 2. Using JDBC Statement Objects to Execute SQL
         * https://dev.mysql.com/doc/connector-j/8.0/en/connector-j-usagenotes-statements.html
         */
        Statement stmt = null;
        ResultSet rs = null;

        try {
            stmt = conn.createStatement();
            rs = stmt.executeQuery("SELECT * FROM zhw.user_info");
            while (rs.next()) {
                System.out.println("name:" + rs.getString("name"));
            }

            // or alternatively, if you don't know ahead of time that
            // the query will be a SELECT...
            if (stmt.execute("update zhw.user_info set gmt_modified = now() where name = '张伟'")) {
                rs = stmt.getResultSet();
                // do something with rs
            } else {
                System.out.println(stmt.getUpdateCount());
            }
            conn.commit();
            // Now do something with the ResultSet ....
        } catch (SQLException ex) {
            // handle any errors
            System.out.println("SQLException: " + ex.getMessage());
            System.out.println("SQLState: " + ex.getSQLState());
            System.out.println("VendorError: " + ex.getErrorCode());
            conn.rollback();

        } catch (Throwable e) {
            System.out.println(e.getMessage());
            conn.rollback();

        } finally {
            // it is a good idea to release
            // resources in a finally{} block
            // in reverse-order of their creation
            // if they are no-longer needed
            if (rs != null) {
                try {
                    rs.close();
                } catch (SQLException ignore) {
                } // ignore
            }
            if (stmt != null) {
                try {
                    stmt.close();
                } catch (SQLException ignore) {
                } // ignore
            }
        }
    } catch (SQLException e) {
        e.printStackTrace();
    }

}
```