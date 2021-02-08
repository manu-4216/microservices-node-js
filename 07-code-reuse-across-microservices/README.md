# Code reuse

Teacher code: https://github.com/StephenGrider/ticketing

We will have a shared lib:

- custom error system
- auth middleware
- request validation middleware

However note that this is only sharable across the same programming language (Node.js in this case).

Best way to do so is with a scoped (private) `npm` package.
