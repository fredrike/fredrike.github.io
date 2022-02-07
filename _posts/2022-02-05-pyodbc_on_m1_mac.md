---
title: Installing pyodbc with Microsoft SQL Server driver on a M1 Mac
last_modified_at: 2022-02-07 13:47:12 +0100
---

The following is a breif scratchpad on the steps taken to install and get pyodbc to work with a Microsoft SQL Server connection on a M1 Mac.

All commands assumes that HomeBrew is installed and working.

Install odbc connections:

```shell
HOMEBREW_NO_ENV_FILTERING=1 ACCEPT_EULA=Y brew reinstall msodbcsql17 mssql-tools unixodbc

# Fix missing link, version might vary
brew link --overwrite mssql-tools@14.0.6.0
brew link --overwrite unixodbc
```

Building pyodbc requires `CPPFLAGS` and `LDFLAGS` to be defined:

```shell
# Set linkflags to build and install pyodbc
CPPFLAGS="-I$(brew --prefix unixodbc)/include" LDFLAGS="-L$(brew --prefix unixodbc)/lib" python3 -m pip install --force --no-cache pyodbc
```

There is known issues with [Microsoft's ODBC driver for SQL Server](https://docs.microsoft.com/en-us/sql/connect/odbc/linux-mac/known-issues-in-this-version-of-the-driver?view=sql-server-ver15#connectivity) using the wrong SSL library, fix this:

```shell
# Fix issues with openssl
rm -rf $(brew --prefix)/opt/openssl
version=$(ls $(brew --prefix)/Cellar/openssl@1.1 | grep "1.1")
ln -s $(brew --prefix)/Cellar/openssl@1.1/$version $(brew --prefix)/opt/openssl
```

After the installation of pyodbc and fixing issues with openssl the guide from Microsoft, [Proof of concept connecting to SQL using pyodbc](https://docs.microsoft.com/en-us/sql/connect/python/pyodbc/step-3-proof-of-concept-connecting-to-sql-using-pyodbc), worked fine.
