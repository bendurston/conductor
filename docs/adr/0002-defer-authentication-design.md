# Defer concrete authentication design until service integration

The requirements deliberately do not pin down how Conductor authenticates to each Service. We know
only that every supported Service needs some method or methods of authentication, and that a
Transfer requires credentials configured for both its Source and Destination. The concrete shape —
OAuth flows, token storage and refresh, first-run login — differs per Service and is easy to get
wrong when guessed in the abstract. We commit to those specifics when we build each Service's
connector and can verify them against the live API, not before.
