### AMQP client in Postgres

Found [this little thing](https://github.com/omniti-labs/pg_amqp) by accident - this is an AMQP library for Postgres. Gives Postgres
the ability to publish AMQP messages as a part of a database transaction. Not very closely related to nginn-messagebus, but helps you
achieve similar goal - sending of messages integrated with db transaction.
