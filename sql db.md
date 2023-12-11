# Database design (SQL)
The SQL database is used to persistently store accepted jobs.
It will contain meta information such as job id, orchestrator task(s) id and all information that was submitted as
arguments to the job.

Any and all information in the SQL database is removed once the result or error is communicated back through
the broker to the frontend.

While there are many choices for a SQL (and even noSQL) database, the choice is made for PostgreSQL because:
- Already using TimescaleDB which is an extension on PostgreSQL. This allows for harmony between technologies in the
  same architecture.
- SQL database over noSQL database due to schema consistencies and transactional guarantees. We are not storing a lot
  of data in the database but we require each entry to have the same structure and we need to guarantees when
  altering one job from multiple threads/components.
