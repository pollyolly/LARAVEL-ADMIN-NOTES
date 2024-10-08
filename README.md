### LARAVEL-ADMIN-NOTES

### LARAVEL ADMIN INSTALLATION
```
https://laravel-admin.org/docs/en/#Installation
```
Install composer
```
https://www.digitalocean.com/community/tutorials/how-to-install-and-use-composer-on-ubuntu-20-04
```
PHP Extensions
```
BCMath PHP Extension
Ctype PHP Extension
Fileinfo PHP extension
JSON PHP Extension
Mbstring PHP Extension
OpenSSL PHP Extension
PDO PHP Extension
Tokenizer PHP Extension
XML PHP Extension
```
Install laravel 5.5
```
composer create-project --prefer-dist laravel/laravel qr_ticket_management "5.5.*"
```
Configure Database
```
cd qr_ticket_management
vi qr_ticket_management/.env

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=mydatabase
DB_USERNAME=myusername
DB_PASSWORD=mypassword
```
Install laravel admin
```
composer require encore/laravel-admin
```
Publish assets and config：
```
php artisan vendor:publish --provider="Encore\Admin\AdminServiceProvider"
```
Finish install
```
php artisan admin:install
```
Reconfigure Laravel Admin
```
cd qr_ticket_management
$php artisan storage:link
$php artisan key:generate
$php artisan cache:clear
$php artisan migrate
$chmod -R 775 storage
$composer dump-autoload
$chmod -R 775 bootstrap/cache
```

Creating Controller and Model
```
Model
$php artisan make:model Models/QRTickets -m

Controller
$php artisan admin:make QRTicketsController --model='App\Models\QRTickets'
```
Updating Columns Database (Go to Latest migration /database/migration/)
```
    public function up()
    {
        Schema::create('q_r_tickets', function (Blueprint $table) {
            $table->increments('id');
            $table->string('ticket_number');
            $table->string('ticket_subject');
            $table->text('ticket_content');
            $table->integer('user_id')->unsigned();
            $table->foreign('user_id')->references('id')->on('users');
            $table->timestamps();
        });
    }
    
$php artisan migrate
```
Important Default Locations
```
Laravel Admin: 
appfolder/app/database/migration/ - Laravel Database Schema
appfolder/app/                    - Admin Models
appfolder/app/Admin/Controllers/  - Admin Controllers
appfolder/app/Admin/routes.php    - Admin Routings
appfolder/config/admin.php        - Extensions Configs
appfolder/config/filesystems.php  - Laravel File Upload Settings
appfolder/.env                    - Laravel DB, SMTP, etc settings

Main Laravel Admin:
appfolder/vendor/encore/laravel-admin/

Laravel:
appfolder/app/Http/Controllers/    - Laravel Client Controllers
appfolder/app/Models/              - Manually created Folder for Models
appfolder/resources/views          - Laravel blades / Views
appfolder/config/session.php       - Laravel Session
appfolder/routes/web.php
```
Move Model to Models Folder
```
0. Seach the Model on config, app/Admin/, appfolder/database/, app/Http/, appfolder/config/, app/Models/
$egrep -r "Model Class"

1. Update app/Model
From: namespace App; 
to: namespace App\Models;

2. Update app/Admin/Controller (app/Http/Controllers/Auth/ and app/Admin/Controllers/)
From: use App\QRTickets; 
to: use App\Models\QRTickets;

3. Update app/config/auth.php
From: 'model' => App\User::class, 
to: 'model' => App\Models\User::class,

4. Update app/database/factories
From: $factory->define(App\User::class 
to: $factory->define(App\Models\User::class

5. Update app/config/services.php
From: 'model' => App\User::class, 
to: 'model' => App\Models\User::class,

6. Run
$composer dump-autoload
$chown -R www-data:www-data appfolder

To Test:
$php artisan make:model Models\Test -m
```
Delete Model
```
1. Delete the Model
2. Delete the Model migration in /database/migration
3. This will Remove All the Data in Database
   $php artisan migrate:refresh 
```
Update Database Schema
```
1. Create Migration
$php artisan make:migration add_fields_to_users_table --table=users

2. Add the code to /database/migration/add_fields_to_users_table.php
Schema::table('users', function ($table) {
     $table->string('student_number');
});

3. Run Migrate
  $php artisan migrate
  or
  $php artisan migrate --path=/database/migrations/migration_q_r_tickets_table.php
```
[Optional: Adding Sessions](https://laravel.com/docs/9.x/session#configuration)
```vim
From Database:
$php artisan session:table
$php artisan migrate
$cd appfolder/.env
SESSION_DRIVER=database
```
[Adding Middleware](https://laravel.com/docs/5.5/middleware#assigning-middleware-to-routes)
```vim
$php artisan make:middleware CheckLoginSession
//app/Http/Middleware/CheckLoginSession.php
public function handle($request, Closure $next)
    {
        if(!$request->session()->get('authID')){
                return redirect('/student-login');
        }
        return $next($request);
    }

//Update: $routeMiddleware
//app/Http/Kernel.php
protected $routeMiddleware = [
     'check.login.session' => \App\Http\Middleware\CheckLoginSession::class
]

//Use in routes/Web.php
Route::get('student-ticket-list', 'StudentSearchController@ticketIndex')
->name('student.ticket.list')
->middleware('check.login.session');

//Process
//First call the Middleware -> then the Router
```
Setup Nginx
```nginx
server {
        listen 80;
        listen [::]:80;
        root /var/www/html/qr_ticket_management/public;
        index index.php index.html;
        server_name _;
        
        add_header X-Frame-Options "SAMEORIGIN";
        add_header X-XSS-Protection "1; mode=block";
        add_header X-Content-Type-Options "nosniff";
        charset utf-8;
        
        location / {
                try_files $uri $uri/ /index.php?$query_string;
        }
        
        location = /favicon.ico { access_log off; log_not_found off; }
        location = /robots.txt  { access_log off; log_not_found off; }
        
        location ~ \.php$ {
                #NOTE: You should have "cgi.fix_pathinfo = 0;" in php.ini
                include fastcgi_params;
                fastcgi_intercept_errors on;
                fastcgi_pass unix:/run/php/php7.4-fpm.sock;
                fastcgi_param  SCRIPT_FILENAME $document_root$fastcgi_script_name;
                fastcgi_buffers 4 16k;
                fastcgi_buffer_size 16k;
        }
        
        location ~ /\.(?!well-known).* {
                deny all;
        }
}
```
Services
```
service nginx start
service mysql start
service php7.4-fpm start
```
### Troubleshooting

Disk [admin] not configured, please add a disk config in `config/filesystems.php`.
```
https://laravel-admin.org/docs/en/model-form-upload
Open 'config/filesystems.php'  add this to the disk array :

        'admin' => [
                'driver' =>'local',
                'root' => public_path('uploads'),
                'visibility' =>'public',
                'url' => env('APP_URL').'/uploads',
        ],
```
Class 'Doctrine\DBAL\Driver\PDOMySql\Driver' not found
```
Check for Laravel 5.5
$composer require doctrine/dbal
or
$composer require doctrine/dbal:2.*
```
Unable to reset Laravel Admin
```
1. Backup Current Laravel Customizations
2. Uninstall laravel-admin
$php artisan admin:uninstall
3. Reinstall Laravel Admin
$composer require encore/laravel-admin
$php artisan vendor:publish --provider="Encore\Admin\AdminServiceProvider"
$php artisan admin:install
```

### Tutorials

[Laravel Admin Walkthrough](https://www.youtube.com/watch?v=F0ujUOAgWqg)

[Install Laravel Admin Panel | Admin Dashboard in Laravel](https://www.youtube.com/watch?v=1-6vBAPvU4k)

[Laravel](https://laravel.com/docs/9.x/eloquent-relationships)

