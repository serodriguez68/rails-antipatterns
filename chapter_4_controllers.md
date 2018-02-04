# Chapter 4 - Controllers
## 4.1 Anti-Pattern: Homemade Authentication
### Solution: Use a well-known library that suits your needs
DON'T make a homemade authentication system unless you are under extreme circumstances.

Here are some names of popular authentication libraries:

* [Devise](https://github.com/plataformatec/devise): Fully featured authentication gem (probably the most well-known gem).
* [Clearance](https://github.com/thoughtbot/clearance): In the middle solution for authentication.
* [Authlogic](https://github.com/binarylogic/authlogic): Minimalistic authentication solution that pushes down the authentication logic to models.
