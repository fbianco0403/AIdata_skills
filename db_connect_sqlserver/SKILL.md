---
name: "db_connect_sqlserver"
description: "Connects to SQL Server databases using pyodbc. Invoke when user provides SQL Server connection info (server, user, password, database) and wants to connect or query data."
---

# SQL Server Database Connection

This skill connects to a SQL Server database using `pyodbc`. When the user provides connection information, follow the steps below to establish and verify the connection.

## Input Format

The user should provide connection info in one of these formats:
- Space-separated: `192.168.39.108 sa SSv2019#tztek Report`
- Key-value: `server=192.168.39.108 user=sa password=SSv2019#tztek database=Report`
- Descriptive: "连接到 192.168.39.108 的 Report 数据库，用户名 sa，密码 xxx"

The parsing order is: **server username password database**

## Steps

### 1. Parse Connection Info

Extract the following four parameters from user input:
- `server`: SQL Server host address (IP or hostname)
- `username`: Database login username
- `password`: Database login password
- `database`: Target database name

If any parameter is missing, ask the user to provide it.

### 2. Check pyodbc Availability

Run `pip list` to check if `pyodbc` is installed. If not, run:
```
pip install pyodbc
```

### 3. Build Connection String

Use the following connection string template:
```python
conn_str = (
    f"DRIVER={{SQL Server}};"
    f"SERVER={server};"
    f"DATABASE={database};"
    f"UID={username};"
    f"PWD={password};"
)
```

If `DRIVER={SQL Server}` fails, try these alternatives in order:
- `DRIVER={ODBC Driver 17 for SQL Server};`
- `DRIVER={ODBC Driver 13 for SQL Server};`
- `DRIVER={SQL Server Native Client 11.0};`

To list available drivers:
```python
import pyodbc
print(pyodbc.drivers())
```

### 4. Test Connection

Connect and verify by:
1. Querying `SELECT @@VERSION` to confirm server version
2. Listing all tables with:
```sql
SELECT TABLE_SCHEMA, TABLE_NAME
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_TYPE='BASE TABLE'
ORDER BY TABLE_SCHEMA, TABLE_NAME
```
3. Print the total table count and table names

### 5. Provide Reusable Connection Code

Generate a connection helper script for the user. Save it as `connect_db.py` in the working directory with this template:

```python
import pyodbc

server = '{server}'
username = '{username}'
password = '{password}'
database = '{database}'

conn_str = (
    f"DRIVER={{SQL Server}};"
    f"SERVER={server};"
    f"DATABASE={database};"
    f"UID={username};"
    f"PWD={password};"
)

def get_connection():
    return pyodbc.connect(conn_str, timeout=10)

if __name__ == '__main__':
    try:
        conn = get_connection()
        print(f"成功连接到 SQL Server: {server}/{database}")
        cursor = conn.cursor()
        cursor.execute("SELECT @@VERSION")
        row = cursor.fetchone()
        print(f"数据库版本: {row[0]}")
        cursor.execute(
            "SELECT TABLE_SCHEMA, TABLE_NAME "
            "FROM INFORMATION_SCHEMA.TABLES "
            "WHERE TABLE_TYPE='BASE TABLE' "
            "ORDER BY TABLE_SCHEMA, TABLE_NAME"
        )
        tables = cursor.fetchall()
        print(f"\n数据库中共有 {len(tables)} 张表:")
        for schema, name in tables:
            print(f"  [{schema}].[{name}]")
        cursor.close()
        conn.close()
        print("\n连接已关闭。")
    except Exception as e:
        print(f"连接失败: {e}")
```

Fill in the actual values for server, username, password, and database.

### 6. Report Results

After successful connection, report:
- Connection status (success/failure)
- Server version
- Total number of tables
- Table list (grouped by schema)

## Important Notes

- Always set a connection timeout (default 10 seconds)
- Never log or expose passwords in output beyond the initial connection script
- Close connections properly after use
- If the connection fails, check network accessibility first, then try alternative ODBC drivers
