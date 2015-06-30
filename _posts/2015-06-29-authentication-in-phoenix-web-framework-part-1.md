---
layout: post
category : webdevelopment
tagline: "Sessions, Flashes, Sign in and Signout with phoenix web framework"
tags : [elixir, phoenix, web frameworks, authentication]
image : "/assets/img/post-bg.jpg"
---

This blog post is supposed to be an easy walkthrough in adding an essential feature in most 
web applications which is Authentication.

### What is Phoenix Web Framework?

Phoenix is a framework for building HTML5 apps, API backends and distributed systems. 
Written in Elixir, you get beautiful syntax, productive tooling and a fast runtime,
from the [phoenix website](www.phoenixframework.org)

### Initial Setup

I won't actually go through lots of details about setting up for phoenix framework.
You can find most of what you will need inside the documentation on their website.

So to install the phoenix framework, just make sure you have elixir installed.

You then have to install hex by typing this command:

	$ mix local.hex

Then you will run this command to install phoenix:

	$ mix archive.install https://github.com/phoenixframework/phoenix/releases/download/v0.13.1/phoenix_new-0.13.1.ez

Also, to create a new phoenix project, ``` $ mix phoenix.new projectname ```


### Real Deal

So after you have created the project, your directory structure will look like this: 
	
	├-- _build
	├-- config
	├-- deps
	├-- lib
	├-- priv
	├-- test
	├-- web
	 
We will use an elixir library called ``` comeonin ``` to help with password hashing. so we will have to add that to our dependencies in the mix.exs file and run the command:


	$ mix do deps.get, compile


Next we will generate our model for the user, so we will use the phoenix generator by running this command: 

	$ mix phoenix.gen.model User users email:string password:string

That will generate the User model which you will find in the `web/models` directory and also a migration which can also be found in `priv/repo/migrations`. You can now run 
	
	$ mix ecto.migrate

to create the table and fields in the database.

So now to the main authentication file.

Navigate to your ```lib/{project_name}/``` folder, and create a new file. You can call it authenticate.ex..

This file will be our main file for checking password hashing, checking if user exists and giving permissions.

Lets first create the module:
	
	defmodule {project_name}.Authenticate do
		
	end

Lets import some important stuffs which will be used by our functions:
	
	alias Comeonin.Bcrypt # for password hashing
	alias {project_name}.User # The module for your user model
	alias {project_name}.Repo
	import Plug.Conn
	import {project_name}.Contacts.Router.Helpers
	import Phoenix.Controller

So the first function we will right will be used to check if hashed and non-hashed passwords match. It takes the user and the normal password as arguments, hashes the password and checks it with the stored one from the user.
	

	def password(user, password) do
       check_password =
        password
     	|> Bcrypt.checkpw(user.password)

        _password(check_password, user)
    end
    defp _password(false, _),   do: {:error, "Please enter a valid email and password"}
    defp _password(true, user), do: {:ok, user}



The next function will be the session function, which checks if an email is in the users table.
	
	def session(users, email) do
	   email_check =
	   users
	   |> Enum.map(fn user -> user.email end)
	   |> Enum.member?(email)

	   case email_check do
	      false -> {:error, "Unauthorized access attempt!"}
	      true  -> {:ok,    "User access granted!"}
	    end
	end

We then write the letmein function that will also check the email in the database before authenticating the user with the password.
		
	def letmein(email, password) do
	   user = Repo.get_by(User, email: email)

	   _letmein(user, password)
	end
	defp _letmein(nil, _) do
	   Comeonin.Bcrypt.dummy_checkpw
	   {:error, "Please enter a valid email and password"}
	end
	defp _letmein(user, password), do: password(user, password)

Lets then write another function, geez too much functions, called login_required that takes an email, checks wether the email is in the database, by calling the session functions. A ```case...do```, which allows us to compare a value against some patterns to find the matching one, handles the email check.
	
	def login_required(conn, nil), do: unauthorized(conn)
	def login_required(conn, ""),  do: unauthorized(conn)
	def login_required(conn, email) do
	   users = Repo.all(User)
	   new_session = session(users, email)

	   case new_session do
	      {:ok, message} -> IO.puts(message)
	        current_user = get_session(conn, :user)
	        assign(conn, :user, current_user)
	      {:error, message} -> IO.puts(message)
	      conn
	      |> redirect to: user_path(conn, :get_signin)
	    end
	end

Another function that handles error messages from the login_required function
	
	def unauthorized(conn) do
      IO.puts("Unauthorized access attempt!")
      conn
      |> fetch_flash
      |> put_flash(:error, "Unauthorized!")
      |> redirect to: user_path(conn, :get_signin)
    end

Whew, Finally a function called authenticate_required to perform as some form of a decorator for all actions of the controllers that needs login permission.
	
	def authenticate_required(conn, _params) do
	   session_email = get_session(conn, :email)
	   conn |> login_required(session_email)
	end

This will be the end of the authenticate.ex file.