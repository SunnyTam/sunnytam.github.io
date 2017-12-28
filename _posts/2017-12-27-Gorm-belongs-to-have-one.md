---
layout: post
current: post
cover:  assets/images/golang.jpg
navigation: True
title: Study on using both BelongsTo and Have One Relationship in Gorm
date: 2017-12-28 14:45:00
tags: [golang]
class: post-template
subclass: 'post tag-golang'
author: sunny
categories: sunny
---

# Study on using both BelongsTo and Have One Relationship in Gorm

## The Problem using both BelongsTo and HaveOne

Base on offical Gorm website, you can define your **BelongsTo** relationship like this
~~~golang
// `User` belongs to `Profile`, `ProfileID` is the foreign key
type User struct {
  gorm.Model
  Profile   Profile
  ProfileID int
}

type Profile struct {
  gorm.Model
  Name string
}

db.Model(&user).Related(&profile)
//// SELECT * FROM profiles WHERE id = 111; // 111 is user's foreign key ProfileID
~~~

And if you want to have **HaveOne** relationship. you can do this

~~~golang
// User has one CreditCard, UserID is the foreign key
type User struct {
    gorm.Model
    CreditCard   CreditCard
}

type CreditCard struct {
    gorm.Model
    UserID   uint
    Number   string
}

db.Model(&user).Related(&card)
~~~

But the offical website do not tell you how to do **BelongsTo** and **HaveOne** together cause if you doing like this will trigger a panic in golang due to the infinity recursive struct. `User` will try to have a `Profile` Struct and the inner `Profile` will contain `User` struct 

~~~golang
type User struct {
  gorm.Model
  Profile   Profile
  ProfileID int
}

type Profile struct {
  gorm.Model
  Name string
  User  User
}

db.Model(&user).Related(&user.Profile)
~~~

## Track to use BelongsTo and HaveOne together

The track you need to do is use pointer in both `User` and `Profile` struct. This pointer avoid an infinity recursive struct 
~~~golang
type User struct {
  gorm.Model
  Profile   *Profile  //using pointer instead
  ProfileID int
}

type Profile struct {
  gorm.Model
  Name string
  User  *User //using pointer instead
}

user.Profile = &Profile{} // let the pointer point to a new Profile
db.Model(&user).Related(user.Profile) // pass the pointer to related directly
user.Profile.User = &user // this will create a reference back to the Profile object. 

db.Save(&user) // this will trigger a panic from infinity recursive update
~~~

This will work fine in get query. But this will create an infinity update between `Profile` and `User`. My solution is to `gorm:"save_associations:false"` after one field. For example.

~~~golang
type User struct {
  gorm.Model
  Profile   *Profile
  ProfileID int
}

type Profile struct {
  gorm.Model
  Name string
  User  *User `gorm:"save_associations:false"`
}

user.Profile = &Profile{} 
db.Model(&user).Related(user.Profile)
user.Profile.User = &user

db.Save(&user) // this nested save will work. Saving both user and profile
db.Save(&user.Profile) // this save will work with only saving profile data
~~~