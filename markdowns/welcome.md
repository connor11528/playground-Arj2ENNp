# Build a Task List with Laravel 5.4 and Vue 2

![](https://cdn-images-1.medium.com/max/800/1*1cxvzNwx9AaQkh9gYC4bzw.jpeg)

We’re going to follow along with
[@felicianopj](https://github.com/felicianopj/laravel-vuejs-tasks) post that is
available
[here](http://felicianoprochera.com/simple-task-app-with-laravel-5-3-and-vuejs/).

#### Create application

I have a short shell script in my Projects folder that I run to create fresh
Laravel apps ([link to
gist](https://gist.github.com/connor11528/fcfbdb63bc9633a54f40f0a66e3d3f2e)) ⭐

So run the script like:

    $ ./create_laravel_app.sh laravel-vue-tasks
    $ cd laravel-vue-tasks

#### Database stuff 📊

We have a Laravel app and we need to store our tasks in the database.

This one artisan command creates **the** **model**, **a migration file** and a
**resource controller**.

    $ php artisan make:model Task -mr

In the model file we need to tell Laravel that the attributes of the database
are “mass assignable”. To do this and turn off some of Laravel’s security by
default functionality we set $guarded to an empty array like so:

**app/Task.php**

    <?php

    namespace App;

    use Illuminate\Database\Eloquent\Model;

    class Task extends Model
    {
        protected $guarded = [];
    }

Also, create a MySQL database on your local machine. In the following commands
we’re going to login to MySQL and run a SQL command that creates a database.

    $ mysql -uroot -p
    MySQL [(none)]> create database laravue;

    Query OK, 1 row affected (0.04 sec)

    MySQL [(none)]> Ctrl-C -- exit!

Our data structure is going to be mad simple. Tasks will have a body of text,
that’s it. We define this structure in
**database/<timestamp>_create_tasks_table.php**.

    Schema::create('tasks', function (Blueprint $table) {
        $table->increments('id');
        $table->longtext('body');
        $table->timestamps();
    });

In your .env file specify that we want to connect to the database we created.

    DB_CONNECTION=mysql
    DB_HOST=127.0.0.1
    DB_PORT=3306
    DB_DATABASE=laravue
    DB_USERNAME=YOUR_USERNAME_HERE
    DB_PASSWORD=YOUR_PASSWORD_HERE

Migrate the database:

    $ php artisan migrate

If you have issues try clearing the cache using php artisan config:clear before
the migration command.

#### Create the API 📡

First we’ll set up routes. Head into **routes/web.php **and we’ll create a route
group that will have all our CRUD actions for tasks. Add this line:

    Route::get('/', function () {
        return view('welcome');
    });

    Route::prefix('api')->group(function() {
        Route::resource('tasks', 'TaskController');
    });

To check out the routes available to your application in Laravel you can always
run: php artisan route:list

Head to **app/Http/Controllers/TaskController.php** and define our routes. It’s
worth noting that we are using **route model binding **so a lot of the heavy
lifting is defined for us. In this instance Laravel takes advantage of
convention over configuration.

    <?php

    namespace App\Http\Controllers;

    use App\Task;
    use Illuminate\Http\Request;

    class TaskController extends Controller
    {
        public function index()
        {
            return Task::latest()->get();
        }

        public function store(Request $request)
        {
            $this->validate($request, [
                'body' => 'required|max:500'
            ]);

            return Task::create([ 'body' => request('body') ]);
        }

        public function destroy($id)
        {
            $task = Task::findOrFail($id);
            $task->delete();
            return 204;
        }
    }

#### Vue.js and Making Things Look Pretty in the Browser™️

Open a new terminal window and run

    $ yarn run watch

This will compile our javascript files, watch for changes and re-update them
anytime we save a file. We now have two terminals going, one with *php artisan
serve* and the other with the *yarn run watch* command. Head over to
[http://localhost:8000/](http://localhost:8000/) to see the Laravel welcome
page.

Our app will load one html file — **resources/views/welcome.blade.php:**

    <!doctype html>
    <html lang="{{ app()->getLocale() }}">
        <head>
            <meta charset="utf-8">
            <meta http-equiv="X-UA-Compatible" content="IE=edge">
            <meta name="viewport" content="width=device-width, initial-scale=1">

            <title>Laravel Vue Task App</title>

            <!-- CSRF Stuff -->
            <meta name="csrf-token" content="{{ csrf_token() }}">
            <script>window.Laravel = { csrfToken: '{{ csrf_token() }}' }</script>

            <!-- Fonts -->
            <link href="{{ asset('css/app.css') }}" rel="stylesheet" type="text/css">

            <!-- Styles -->
            <link rel="stylesheet" href="
    " integrity="sha384-BVYiiSIFeK1dGmJRAkycuHAHRg32OmUcww7on3RYdg4Va+PmSTsz/K68vbdEjh4u" crossorigin="anonymous">

       </head>
       <body>
            <div class="container" id='app'>
                <task-list></task-list>
            </div>

            <!-- Scripts -->
            <script src="{{ asset('js/app.js') }}"></script>
        </body>
    </html>

Notice we add a CSRF field so Laravel can validate our requests. We’re also
using Bootstrap 3 for styling. We load in our javascript and have a component
called task list. We can define that in
**resources/assets/js/components/TaskList.vue.**

In the Vue file we’ll do all our Javascripts magic and make HTTP requests using
axios (comes with Laravel 5.4 by default).

    <template>
        <div class='row'>
            <h1>My Tasks</h1>
            <h4>New Task</h4>
            <form action="#" @submit.prevent="createTask()">
                <div class="input-group">
                    <input v-model="task.body" type="text" name="body" class="form-control" autofocus>
                    <span class="input-group-btn">
                        <button type="submit" class="btn btn-primary">New Task</button>
                    </span>
                </div>
            </form>
            <h4>All Tasks</h4>
            <ul class="list-group">
                <li v-if='list.length === 0'>There are no tasks yet!</li>
                <li class="list-group-item" v-for="(task, index) in list">

                     {{ task.body }}

                     <button @click.prevent="deleteTask(task.id)" class="btn btn-danger btn-xs pull-right">Delete</button>
                </li>
            </ul>
        </div>
    </template>

    <script>
        export default {
            data() {
                return {
                    list: [],
                    task: {
                        id: '',
                        body: ''
                    }
                };
            },
            
            created() {
                this.fetchTaskList();
            },
            
            methods: {
                fetchTaskList() {
                    axios.get('api/tasks').then((res) => {
                        this.list = res.data;
                    });
                },
     
                createTask() {
                    axios.post('api/tasks', this.task)
                        .then((res) => {
                            this.task.body = '';
                            this.edit = false;
                            this.fetchTaskList();

                        })
                        .catch((err) => console.error(err));
                },
     
                deleteTask(id) {
                    axios.delete('api/tasks/' + id)
                        .then((res) => {
                            this.fetchTaskList()
                        })
                        .catch((err) => console.error(err));
                },
            }
        }
    </script>
    </script>

Load our component into the main Javascript app file. That is located in
**resources/assets/js/app.js**

Add this line:

    Vue.component('task-list', require('./components/TaskList.vue'));

That’s it! We’re done. The source code is available [on
github](https://github.com/connor11528/laravel-vue-tasks).

![](https://camo.githubusercontent.com/55dd2b124cfaf8e144982c64a47581d830dc785d/68747470733a2f2f63646e2d696d616765732d312e6d656469756d2e636f6d2f6d61782f3830302f312a4b34716c3534445265416538626775553072545874512e706e67)

<hr>

Originally published [on Medium](https://medium.com/@connorleech/build-a-task-list-with-laravel-5-4-and-vue-2-9be0bba06801)

