# Laravel-Best-Practises

## Controller code
### Bad
```
class UserController extends Controller
{
    public function index()
    {
        // Query the database directly in the controller
        $users = DB::table('users')->get();

        return view('users.index', compact('users'));
    }

    public function store(Request $request)
    {
        // No validation, direct database insertion
        DB::table('users')->insert([
            'name' => $request->input('name'),
            'email' => $request->input('email'),
            'password' => bcrypt($request->input('password')),
        ]);

        return redirect()->route('users.index');
    }
}
```
#### Problems
- Direct database queries in controllers are not clean and violate the separation of concerns.
- Missing validation for incoming data.
- No use of Eloquent, which is one of Laravel's core strengths.

### Good
```
class UserController extends Controller
{
    public function index()
    {
        // Use Eloquent model to fetch users
        $users = User::all();

        return view('users.index', compact('users'));
    }

    public function store(UserRequest $request)
    {
        // Validation logic is handled through a custom Form Request class
        User::create([
            'name' => $request->name,
            'email' => $request->email,
            'password' => bcrypt($request->password),
        ]);

        return redirect()->route('users.index');
    }
}
```
#### Improvements
- Using Eloquent ORM for cleaner, more readable queries.
- Data validation is handled by a custom `UserRequest` form request, improving code separation and reusability.

## Direct SQL Queries in Controllers 
### Bad 
```
public function index()
{
    $users = DB::select('SELECT * FROM users');
    return response()->json($users);
}
```
#### Problems
- Using raw SQL in the controllers violates the MVC principle, making code less readable and harder to maintain.
- There's no use of Laravel's Eloquent ORM, which provides more readable and safer database interactions.

### Good
```
use App\Models\User;

public function index()
{
    $users = User::all();
    return response()->json($users);
}
```
#### Improvements
- Uses Eloquent ORM, which improves readability and abstracts database operations.
- Cleanerand more readable code.

## Database Querying
### Bad
```
// Direct query execution without optimization
$users = DB::table('users')
    ->join('orders', 'users.id', '=', 'orders.user_id')
    ->where('orders.created_at', '>', now()->subMonth())
    ->select('users.name', 'orders.total')
    ->get();
```
#### Problems
- Direct query construction in controllers or views can lead to maintenance issues.
- No use of eager loading for relationships, potentially causing the N+1 problem.
### Good
```
class UserController extends Controller
{
    public function index()
    {
        // Use Eloquent relationships and eager loading
        $users = User::with('orders')->whereHas('orders', function($query) {
            $query->where('created_at', '>', now()->subMonth());
        })->get();

        return view('users.index', compact('users'));
    }
}
```
#### Improvements
- Utilised eager loading (`with()`) to prevent the N+1 query problem.
- Encapsulated business logic within the model or controller, following the single responsibility principle.

## Validation
### Bad
```
public function store(Request $request)
{
    // No validation, using raw request data
    $user = new User;
    $user->name = $request->input('name');
    $user->email = $request->input('email');
    $user->password = bcrypt($request->input('password'));
    $user->save();
}
```
#### Problems
- Missing validation logic.
- Direct manipulation of the request data without filtering or validating inputs.
### Good
```
class UserRequest extends FormRequest
{
    public function rules()
    {
        return [
            'name' => 'required|string|max:255',
            'email' => 'required|email|unique:users,email',
            'password' => 'required|string|min:8|confirmed',
        ];
    }
}
```
Controller
```
public function store(UserRequest $request)
{
    $user = User::create($request->validated());
}
```
#### Improvements
- Custom form request `UserRequest` handles validation.
- The `validated()` method ensures only validated data is passed to the model.

## Security Concerns
### Bad 
```
public function login(Request $request)
{
    $user = User::where('email', $request->email)->first();

    if ($user && Hash::check($request->password, $user->password)) {
        Auth::login($user);
        return redirect()->route('dashboard');
    }

    return back()->withErrors(['email' => 'Invalid credentials']);
}
```
#### Problems
- While the logic is mostly correct, it does not utilize Laravel's built-in `attempt()` method, which can handle additional features such as rate-limiting and locking.

### Good
```
public function login(Request $request)
{
    $credentials = $request->only('email', 'password');

    if (Auth::attempt($credentials)) {
        return redirect()->route('dashboard');
    }

    return back()->withErrors(['email' => 'Invalid credentials']);
}
```
#### Improvements
- Using `Auth::attempt()` method for authentication, which provides additional security features.
- `attempt()` automatically hashes the password and checks for other security measures.

## Error Handling
### Bad
```
public function show($id)
{
    $user = User::find($id);

    if (!$user) {
        return response('User not found', 404);
    }

    return view('users.show', compact('user'));
}
```
#### Problems
- Returns a raw response instead of using Laravel’s built-in exception handling.
- Repeating error message logic in multiple places.
### Good
```
public function show($id)
{
    $user = User::findOrFail($id);

    return view('users.show', compact('user'));
}
```
#### Improvements
- Use `findOrFail()`, which automatically throws a `ModelNotFoundException` if the user is not found.
- Laravel will catch the exception and render a 404 page automatically, providing consistent error handling.

## File Uploads
### Bad
```
public function upload(Request $request)
{
    $request->file('image')->move('uploads', 'image.jpg');
}
```
#### Problems
- Hardcoding file paths and names.
- Lack of validation for file types, sizes, etc.
### Good
```
public function upload(Request $request)
{
    $request->validate([
        'image' => 'required|image|mimes:jpeg,png,jpg,gif|max:2048',
    ]);

    $path = $request->file('image')->store('uploads', 'public');

    return response()->json(['path' => $path]);
}
```
#### Improvements
- Validate file type and size before uploading.
- Store files using Laravel's built-in storage system for better handling of file paths.

## User model 
### Full name
### Bad
```
public function getFullNameLong(): string
{
    return 'Mr. ' . $this->first_name . ' ' . $this->middle_name . ' ' . $this->last_name;
}
```
#### Problems
- Hardcoding the title "Mr." in the `getFullNameLong` method is not flexible. It assumes that every name is male and doesn’t allow flexibility for different titles (e.g., Mrs., Dr., etc.) or genders.
### Good
```
public function getFullNameLong(): string
{
    return $this->title . ' ' . $this->first_name . ' ' . $this->middle_name . ' ' . $this->last_name;
}
```
#### Improvements
- Titles should be dynamic and flexible. If you need to address someone based on their gender or role (Mr., Mrs., Dr.), you should store it as an attribute of the user or pass it dynamically to the method.

### Better
```
public function getFullNameLong(): string
{
    return $this->title . ' ' . ($this->first_name ?? '') . ' ' . ($this->middle_name ?? '') . ' ' . ($this->last_name ?? '');
}
```
- Ensure that null values are handled properly. You can use conditional checks or the null coalescing operator (`??`) to handle missing values.
- This will safely return an empty string for any missing part, but it might not be the ideal solution for all cases (you may want better error handling, like a fallback string).

### Short name
### Bad
```
public function getFullNameShort(): string
{
    return $this->first_name[0] . '. ' . $this->last_name;
}
```
#### Problems
- In the `getFullNameShort` method, the short name is formed by only the first letter of the first name (`$this->first_name[0]`). This is not very robust, as it assumes the first name is always at least one character long.
### Good
```
public function getFullNameShort(): string
{
    $firstNameInitial = !empty($this->first_name) ? $this->first_name[0] . '.' : '';
    return $firstNameInitial . ' ' . $this->last_name;
}
```
#### Improvements
- Add a check to ensure the first name is non-empty and handle edge cases where the first name may be missing or empty.

## Hardcoding configuration values
### Bad 
```
public function sendEmail()
{
    $to = 'example@example.com';
    $subject = 'Hello World';
    mail($to, $subject, 'This is a test email.');
}
```
#### Problems
- Email details are hardcoded, making the code inflexible and difficult to maintain.
- It doesn't utilize Laravel's mail configuration.

### Good
```
use Illuminate\Support\Facades\Mail;

public function sendEmail()
{
    Mail::to(config('mail.default_to_address'))->send(new App\Mail\WelcomeMail());
}
```
#### Improvements
- Email recipients and settings are fetched from the configuration files (`config/mail.php`).
- Using Laravel's `Mail` facade integrates better with SMTP, services like MailGun.

## Blade Files
### Direct querying in Blade files
### Bad
```
<h1>Users</h1>
@foreach (User::all() as $user)
    <p>{{ $user->name }}</p>
@endforeach
```
#### Problems
- Direct querying the database inside a Blade template is bad practice.
- It tightly couples the view and database logic

### Good 

Controller
```
public function index()
{
    $users = User::all();
    return view('users.index', compact('users'));
}
```
Blade
```
<h1>Users</h1>
@foreach ($users as $user)
    <p>{{ $user->name }}</p>
@endforeach
```
#### Improvements
- Separation of concerns: The controller handles fetching the data, and the Blade template focuses only on displaying it.
- Improved readability and maintainability.

### Using `echo` in Blade files
### Bad
```
<p><?php echo $user->name; ?></p>
```
#### Problems
- It’s verbose and does not take advantage of Laravel's Blade template engine.

### Good
```
<p>{{ $user->name }}</p>
```
### Even better
```
<p>{{ $user->name ?? 'Guest' }}</p>
```
#### Improvements
- Blade is more concise, readable, and automatically escapes output to prevent XSS attacks. The fallback ensures proper defaults.

## Not using Route Model Binding
### Bad
```
public function show($id)
{
    $user = User::find($id);
    if (!$user) {
        abort(404);
    }
    return view('user.show', compact('user'));
}
```
#### Problems
- Manual `find()` checks and error handling for invalid IDs.

### Good
```
public function show(User $user)
{
    return view('user.show', compact('user'));
}
```
#### Improvements
- Laravel's Route Model Binding automatically fetches the `User` by ID.
- If the user is not found, a 404 response is automatically retruned.
`Route::get('/users/{user}', [UserController::class, 'show']);`

## Hardcoding Dependencies instead of using Dependency Injection
### Bad
```
public function sendNotification()
{
    $mailer = new \App\Services\Mailer();
    $mailer->send('Hello World');
}
```
#### Problems
- Hardcoding dependencies makes the code less testable and harder to replace or mock.

### Good
```
use App\Services\Mailer;

public function sendNotification(Mailer $mailer)
{
    $mailer->send('Hello World');
}
```
#### Improvements
- Dependencies are injected into the method or constructor, improving testability.
- Laravel's service container automatically resolves the required dependencies.

## Hardcoding configurations
### Bad
```
$apiKey = '12345'; // API key hardcoded
```
#### Problems
- Hardcoding sensitive data like API keys or configurations makes it difficult to change and insecure if the code is shared.

### Good
```
$apiKey = config('services.api.key');
```
#### Improvements
- Using configuration files centralises sensitive data and allows for environment-specific configurations.

## Using `env()` in code outside of config files
### Bad
```
public function uploadFile()
{
    $path = env('UPLOAD_PATH', 'uploads/default');
    Storage::put($path . '/file.txt', 'content');
}
```
#### Problems
- `env()` should only be used in configuration files, not directly in the application logic.
- It makes testing harder because `env()` is only loaded during runtime.

### Good
Inside `config/filesystems.php`
```
upload_path => env('UPLOADED_PATH', 'uploads/default');
```
Inside controller 
```
public function uploadFile()
{
    $path = config('filesystems.upload_path');
    Storage::put($path . '/file.txt', 'content');
}
```
#### Improvements
- Configuration values are centralised in config files.
- `config()` allows the use of caching, improving performance.

## Mass assignment without guarded fields
### Bad
```
public function store(Request $request)
{
    User::create($request->all());
}
```
#### Problems
- This allos all user input to be mass assigned, making the application vulnerable to mass assign attacks.

### Good
Inside `User` model
```
protected $fillable = ['name', 'email', 'password'];
```
Inside the controller
```
public function store(Request $request)
{
    $data = $request->only(['name', 'email', 'password']);
    $data['password'] = bcrypt(data['password']);

    User::create($data);
}
```
#### Improvements
- The `fillable` property explicitly defines which fields can be mass assigned.
- Prevents unauthorised fileds from being updated maliciously.

## Lack of pagination for large datasets
### Bad
```
public function index()
{
    $users = User::all();
    return response()->json($users);
}
```
#### Problems
- Fetching all records can cause issues for large datasets.

### Good
```
public function index()
{
    $users = User::paginate(10);
    return response()->json($users);
}
```
#### Improvements
- Adds pagination to avoid loading a large dataset into memory.
- Improves application performance.

## Best Practices accepted by 
Laravel has some built in functionality and community packages can help instead of using 3rd party packages and tools.
| Task | Standard Tools | 3rd Party Tools | 
|---|---|---|
| Authorization | Policies | Entrust, Sentinel and other packages |
| Compiling Assests | Laravel Mix, Vite | Grunt, Gulp, and other packages |
| Development Environment | Laravel Sail, Homestead | Docker |
| Deployment | Laravel Forge | Deployer and other solutions | 
| Unit Testing | PHPUnit | Pest |
| Browser Testing | Laravel Dusk | Codeception | 
| DB | Eloquent | SQL, Doctrine |
| Templates | Blade | Twig |
| Working With Data | Laravel Collections | Arrays |
| Form Validation | Request classes | Validation in controller | 
| Authentication | Built In | 3rd party packages, your own solution | 
| API authentication | Laravel Passport, Laravel Sanctum | 3rd party JWT and OAuth packages | 
| Creating an API | Built in | Dingo API or similar |
| Working with DB structure | Migrations | Working directly with the DB |
| Localisition | Built in | 3rd party packages |
| Realtime user interfaces | Laravel Echo, Pusher | 3rd party packages and working with WebSockets directly |
| Generating testing data | Seeder classes, Model Factories, Faker | Creating testing data manually |
| Task scheduling | Laravel Task Scheduler | Scripts and 3rd party packages |
| DB | MySQL, PostgreSQL, SQLite, SQL Server | MongoDB|

## Laravel Naming Conventions 
To follow PSR standards
And, follow naming conventions accepted by the Laravel community: 
| What | How | Good | Bad |
|---|---|---|---|
| Controller | Singular | `ArticaleController` | `ArticalesController` |
| Route | Plural | `articles/1` | `article/1` |
| Route Name | snake_case with dot notation | `users.show_active` | `users.show-active`, `show-active-users` |
| Model | Singular | `User` | `Users` |
| hasOne or belongsTo relationship | Singular | `articleComment` | `articleComments`, `article_comments` |
| All other relationships | Plural | `articleComments` | `articleComment`, `article_comments` | 
| Table | Plural | `article_comments` | `article_comment`, `articleComments` |
| Pivot Table | Singular model names in alphabetical order | `article_user` | `users_article`, `articles_users` |
| Table Column | snake_case without model name | `meta_title` | `MetaTitle`, `article_meta_title` |
| Model Proprty | snake_case | `$model->created_at` | `$model->createdAt` |
| Foreign Key | Singular model name with _id suffix | `article_id` | `ArticleId`, `id_article`, `article_id` |
| Primary Key | - | `id` | `custom_id` | 
| Migration | - | `2017_01_01_000000_create_articles_table` | `2017_01_01_000000_articles` | 
| Method | camelCase | `getAll` | `get_all` |
| Method in resource controller | Table | `store` | `saveArticle` |
| Method in test class | camelCase | `testGuestCannotSeeArticle` | `test_guest_cannot_see_article` | 
| Variable | camelCase | `$articlesWithAuthor` | `$articles_with_author` |
| Collection | Descriptive, Plural | `$activeUsers = User::active()->get()` | `$active`, `$data` |
| Object | Descriptive, Singular | `$activeUser = User::active()->first()` | `$users`, `$obj` | 
| Config and language files index | snake_case | `articles_enabled` | `ArticlesEnabled`, `articles-enabled` |
| View | kabab-case | `show-filtered.blade.php` | `showFiltered.blade.php`, `show_filtered.blade.php` |
| Config | snake_case | `google_calendar.php` | `googleCalendar.php`, `google-calendar.php` |
| Contract (Interface) | Adjective or noun | `AuthenticationInterface` | `Authenticatable`, `IAuthentication` | 
| Trait | Adjective | `Notifiable` | `NotificationTrait` |
| Trait (PSR) | Adjective | `NotifiableTrait` | `Notification` |
| Enum | Singular | `UserType` | `UserTypes`, `UserTypeEnum` |
| Form Request | Singular | `UpdateUserRequest` | `UpdateUserFormRequest`, `UserFormRequest`, `UserRequest` | 
| Seeder | Singular | `UserSeeder` | `UsersSeeder` |
| Language File Names | Lower case, snake_case | `user_management.php`, `order_status.php` | `UserManagement.php`, `OrderStatus.php` |
| Language Files | Lower case, snake_case | `'login_failed'`, `'user'` | `'LoginFailed'`, `'User'` |
