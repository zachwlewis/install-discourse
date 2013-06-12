Create a database dump of the current database.

```bash
$ sudo pg_dump database > outfile -U username
```

Copy the database into the target database. (Note, `infile` should be the file you created previously named `outfile`.)

```bash
$ sudo dropdb 'database' -U username
$ sudo createdb 'database' -U username
$ psql database < infile -U username
```
