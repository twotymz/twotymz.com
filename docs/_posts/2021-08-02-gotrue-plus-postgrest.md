---
layout: post
title: GoTrue + PostgREST
date: 2021-08-02 00:00:00 -0000
---
When PostgREST receives a request that contains a valid JWT with a `role` claim it will switch to a database role with that name for the duration of the HTTP request.  GoTrue doesn't issue JWT's with a `role` claim by default but it's a pretty easy fix to make it do so.

To start, you'll probably want to fork the GoTrue (repo)[https://github.com/netlify/gotrue] so you have a place to make your changes.  Once you have a copy of their baseline you can start editing.

The first change that we need to make is to alter the structure that holds the user claims when GoTrue generates a new access token.  Specifically, we need to add the `Role` claim to the `GoTrueClaims` structure in `api/token.go` like so:

```go
type GoTrueClaims struct {
	jwt.StandardClaims
	Email        string                 `json:"email"`
	AppMetaData  map[string]interface{} `json:"app_metadata"`
	UserMetaData map[string]interface{} `json:"user_metadata"`
	Role         string                 `json:"role"`
}
```

The second change that we need to make is to populate the new `Role` claim in the `generateAccessToken` function in the same file.   You have a choice here as to what you want to populate the claim with.  Our team chose to populate it with the corresponding value from the GoTrue `User` structure.

```go
func generateAccessToken(user *models.User, expiresIn time.Duration, secret string) (string, error) {
	claims := &GoTrueClaims{
		StandardClaims: jwt.StandardClaims{
			Subject:   user.ID.String(),
			Audience:  user.Aud,
			ExpiresAt: time.Now().Add(expiresIn).Unix(),
		},
		Email:        user.Email,
		AppMetaData:  user.AppMetaData,
		UserMetaData: user.UserMetaData,
		Role:         user.Role
	}

	token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
	return token.SignedString([]byte(secret))
}
```

That's it.  Your GoTrue instance should now issue JWT access tokens with a `role` claim that can be used by PostgREST.
