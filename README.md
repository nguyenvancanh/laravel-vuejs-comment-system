# Laravel-vuejs-comment-system

Vẫn nằm trong series những bài viết nghiên cứu về Laravel và Vuejs của cá nhân tôi, Hôm nay tôi xin giới thiệu tới quý bạn đọc cách để tạo chức năng comment cho một bài viết. Một chức năng vô cùng hữu dụng cho những ứng dụng web của bạn, mặc dù nó không realtime như là chatroom, tuy nhiên bạn cũng có thể comment và đọc được comment ngay khi comment mới được tạo.

Chắc hẵn ai trong số chúng ta cũng không còn quá lạ lẫm với chức năng comment nữa. Một chức năng comment hoàn thiện sẽ bao gồm những chức năng chính sau:

- Tạo mới một comment
- Trả lời comment
- Vote comment (Bao gồm tăng vote hoặc giảm vote)
- Báo cáo comment là Spam.

Trong bài viết này, công nghệ được tôi sử dụng tới gồm có:

- Bootstrap
- Loadash
- Laravel 
- Vuejs

## Create Migration Tables

Trước hết, chúng ta cần tạo các bảng cơ sở dữ liệu cho hệ thống.

Bắt đầu với những câu lệnh sau để tạo những file migrate cần thiết

```
php artisan make:migration comments
```

Mở file vừa được tạo và thêm những đoạn code sau vào function Up().

```
        Schema::create('comments', function (Blueprint $table) {
 
           $table->increments('id');
 
           $table->text('comment');
 
           $table->integer('votes')->default(0);
 
           $table->integer('spam')->default(0);
 
           $table->integer('reply_id')->default(0);
 
           $table->string('page_id')->default(0);
 
           $table->integer('users_id');
 
           $table->timestamps();
 
       });
```

Tiếp tới là bảng comment_vote

```
  php artisan make:migration comment_user_vote
```

```
      Schema::create('comment_user_vote', function (Blueprint $table) {
 
           $table->integer('comment_id');
 
           $table->integer('user_id');
 
           $table->string('vote',11);
 
       });
```

Cuối cùng là bảng comment_spam

```
  php artisan make:migration comment_spam
```

```
        Schema::create('comment_spam', function (Blueprint $table) {
 
           $table->integer('comment_id');
 
           $table->integer('user_id');
 
       });
```

Chạy lệnh 

```
  php artisan migrate
```
để tạo các tables từ các file migration bên trên. Tất nhiên là một chức năng comments thì cần có tác nhân thực hiện các hành động đó là người dùng. Chúng ta sẽ sử dụng Laravel auth để  tạo cho hệ thống chức năng quản lý người dùng. 

```
	php artisan make:auth
  
```

Tiếp theo, hãy tạo các Models cho hệ thống bằng câu lệnh:

```
  php artisan make:model Comment
```

Mở file Comment.php và thêm code:

```
<?php
 
namespace App;
 
use Illuminate\Database\Eloquent\Model;
 
class Comment extends Model
 
{
 
   //
 
   /**
 
    * Fillable fields for a course
 
    *
 
    * @return array
 
    */
 
   protected $fillable = ['comment','votes','spam','reply_id','page_id','users_id'];
 
   protected $dates = ['created_at', 'updated_at'];
 
   public function replies()
 
   {
 
       return $this->hasMany('App\Comment','id','reply_id');
 
   }
 
}
```

Tương tự cho những bảng còn lại,

```
  php artisan make:model CommentVote
```

```
namespace App;
 
use Illuminate\Database\Eloquent\Model;
 
class CommentVote extends Model
 
{
 
   //
 
   protected $fillable = ['comment_id','user_id','vote'];
 
   protected $table = "comment_user_vote";
 
   public $timestamps = false;
 
}
```

```
  	
php artisan make:model CommentSpam
```

```
namespace App;
 
use Illuminate\Database\Eloquent\Model;
 
class CommentSpam extends Model
 
{
 
   //
 
   protected $fillable = ['comment_id','user_id'];
 
   protected $table = "comment_spam";
 
   public $timestamps = false;
 
}
```

Tạo controller chính.

```
php artisan make:controller CommentController
```
