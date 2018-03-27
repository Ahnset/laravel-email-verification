# Email verification in a Laravel 5.6 app

This app is going to send emails to verify that an email address is real when users sign up. Before their account is active users need
to click a special link that will verify their account and that they're in control of that email address. This is a common 
web functionality we've all experienced when signing up for new accounts online. The impetus for this project is that I deployed 
a Laravel app for people to use and put it behind a login form. Many people used garbage emails to sign up and access the app. 
This isn't protected by default with Laravel's auth scaffold, so let's add it real quick!

> Draws heavily from [this tutorial](https://hackernoon.com/how-to-use-queue-in-laravel-5-4-for-email-verification-3617527a7dbf) by Ahmed Khan ([@ahmedkhan847](https://github.com/ahmedkhan847))

## Step 1

First step is to create a new Laravel app and generate the generic Laravel auth scaffold:

```
$ php artisan make:auth
```

## Step 2

Head into the Laravel database migrations files and find the **create_users_table** migration. (Hint: it's in **database/migrations**.) 
From there, add two columns to the users table for a token and an is verified field. The verified will default to zero for false.

```
public function up()
{
	Schema::create('users', function (Blueprint $table) {
		$table->increments('id');
		$table->string('name');
		$table->string('email')->unique();
		$table->string('password');
		$table->tinyInteger('verified')->default(0);
		$table->string('email_token')->nullable();
		$table->rememberToken();
		$table->timestamps();
	});
} 
```

## Step 3

Add a queue table for queued jobs and failed jobs, then migrate the database to set it all up.

``` 
$ php artisan queue:table
$ php artisan queue:failed-table
$ php artisan migrate
```

> If you get an error saying access denied, make sure you create a MySQL database for your Laravel app to connect to. You can do that by 
running `mysql -uroot -p` and then the sql command `create database laravelemailverification;`. Download something like [Sequel Pro](https://www.sequelpro.com/) 
to view your tables and database content.

## Step 4

For the migration to work you'll already have to have modified your **.env** file to connect to your database. Now we're going to 
modify it to connect to a queue driver and mail driver. This is going to use the database as a queue driver and gmail for verification
emails.

``` 
QUEUE_DRIVER=database
MAIL_DRIVER=smtp
MAIL_HOST=smtp.gmail.com
MAIL_PORT=587
MAIL_USERNAME=connorleech@gmail.com
MAIL_PASSWORD=
MAIL_ENCRYPTION=tls
MAIL_FROM_ADDRESS=connorleech@gmail.com
MAIL_FROM_NAME="Connor"
```

> For the new values to take effect you may need to run `php artisan config:clear`

## Step 5

Create the email that you want to send to the user. There is a handy artisan command for scaffolding emails:

``` 
$ php artisan make:mail EmailVerification
```

That file is in **app/Mail** and can be modified to look like so: 

``` 
<?php

namespace App\Mail;

use Illuminate\Bus\Queueable;
use Illuminate\Mail\Mailable;
use Illuminate\Queue\SerializesModels;
use Illuminate\Contracts\Queue\ShouldQueue;

class EmailVerification extends Mailable
{
    use Queueable, SerializesModels;
    
    protected $user;

    /**
     * Create a new message instance.
     *
     * @return void
     */
    public function __construct($user)
    {
        $this->user = $user;
    }

    /**
     * Build the message.
     *
     * @return $this
     */
    public function build()
    {
        return $this->view('email.verify_account')->with([
            'email_token' => $this->user->email_token    
        ]);
    }
}
```

## Step 6

Create the email view within **resources/views/email/verify_account.blade.php**:

``` 
<h1>Click the Link To Verify Your Email</h1>
Click the following link to verify your email {{ url('/verifyemail/' . $email_token) }}
```

## Step 7

Now we need to make a new job that will fire off the email:

``` 
$ php artisan make:job SendVerificationEmail
```

That job will live in **app/Jobs**.

