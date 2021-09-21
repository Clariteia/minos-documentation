# CQRS

Command-Query responsibility segregation is an architectural style that aims at separating how commands and queries are
resolved within each service. A `command` is an operation that modifies the internal state of the service, although it
does not return any value. On the other hand, a `query` computes or retrieves any data concerning business logic.
