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

Trong controller này, chung ta sẽ xử lý tất cả các chức năng create comment, get comment, hay update comment. Để tạo comment, tạo function sau trong file controller vừa tạo.

```
public function store(Request $request)
 
   {
 
       $this->validate($request, [
 
       'comment' => 'required',
 
       'reply_id' => 'filled',
 
       'page_id' => 'filled',
 
       'users_id' => 'required',
 
       ]);
 
       $comment = Comment::create($request->all());
 
       if($comment)
 
           return [ "status" => "true","commentId" => $comment->id ];
 
   }
```

Để lấy comment từ database, tạo funciton index trong controller.

```
public function index($pageId)
 
   {
 
       //
 
       $comments = Comment::where('page_id',$pageId)->get();
 
       $commentsData = []
 
       foreach ($comments as $key) {
 
           $user = User::find($key->users_id);
 
           $name = $user->name;
 
           $replies = $this->replies($key->id);
 
           $photo = $user->first()->photo_url;
 
           $reply = 0;
 
           $vote = 0;
 
           $voteStatus = 0;
 
           $spam = 0;
 
           if(Auth::user()){
 
               $voteByUser = CommentVote::where('comment_id',$key->id)->where('user_id',Auth::user()->id)->first();
 
               $spamComment = CommentSpam::where('comment_id',$key->id)->where('user_id',Auth::user()->id)->first();              
 
               if($voteByUser){
 
                   $vote = 1;
 
                   $voteStatus = $voteByUser->vote;
 
               }
 
               if($spamComment){
 
                   $spam = 1;
 
               }
 
           }          
 
           if(sizeof($replies) > 0){
 
               $reply = 1;
 
           }
 
           if(!$spam){
 
               array_push($commentsData,[
 
                   "name" => $name,
 
                   "photo_url" => (string)$photo,
 
                   "commentid" => $key->id,
 
                   "comment" => $key->comment,
 
                   "votes" => $key->votes,
 
                   "reply" => $reply,
 
                   "votedByUser" =>$vote,
 
                   "vote" =>$voteStatus,
 
                   "spam" => $spam,
 
                   "replies" => $replies,
 
                   "date" => $key->created_at->toDateTimeString()
 
               ]);
 
           }       
 
       }
 
       $collection = collect($commentsData);
 
       return $collection->sortBy('votes');
 
   }
```

Trong controller thì chỉ là những thao tác insert vào db, hoặc là lấy data từ trong db ra. Quan trọng nhất trong bài viết này tôi muốn truyền tải tới các bạn là tư tưởng để làm comment system. Khi các bạn đã nắm chắc tư tưởng để làm rồi. đến lúc đó các bạn có thể tủy chỉnh chương trình của bạn theo ý thích. Quá trình học chúng ta cũng không nện dập khuôn, máy móc code của người khác. mà nên đọc, hiểu và code theo cách hiểu của mình. Kỹ thuật sử dụng chính trong bài này là Vuejs, vì vậy tiếp theo chúng ta sẽ chuyển sang nghiên cứu sâu hơn với phần Vuejs của hệ thống.

## Tạo Components

Trước tiên, chúng ta cần tạo mới một file components và đăng ký với hệ thống trong file app.js

```
Vue.component('comment', require('./components/Comments.vue'));
```

Mở file component và thêm code:

```
<template>
    <div class="comments-app">
        <h1>Comments</h1>
        <div class="root-comment-form" v-if="user">
            <!-- Comment Avatar -->
            <div class="row">
                <div class="col-sm-1">
                    <div class="thumbnail-comment">
                        <img :src="user.user_avatar" class="gravatar2">
                    </div>
                </div>
                <form v-on:submit.prevent="saveComment()" class="form-horizontal" id="commentForm" role="form">
                    <div class="form-group">
                        <div class="col-sm-11">
                            <textarea class="form-control" name="addComment" id="addComment" rows="5" placeholder="Add comment..." v-model="message"></textarea>
                        </div>
                    </div>
                    <div class="form-group">
                        <div class="col-sm-offset-2 col-sm-10">
                            <button class="btn btn-success btn-circle text-uppercase" type="submit" style="float:right;"><span class="glyphicon glyphicon-send"></span>Add Comment</button>
                        </div>
                    </div>
                </form>
            </div>
        </div>
        <div class="comment-thread" v-if="comments">
            <div v-for="(comment, index) in commentsData">
                <div class="tab-pane active" id="comments-logout">
                    <ul class="media-list">
                        <li class="media">
                            <a class="pull-left" href="#">
                                <img :src="comment.us_avatar" class="gravatar2">
                            </a>
                            <div class="media-body">
                                <div class="well well-lg">
                                    <h4 class="media-heading text-uppercase reviews"> {{ comment.name }} </h4>
                                    <ul class="media-date reviews list-inline">
                                        <li class="dd">commented {{ comment.time_ago }}</li>
                                    </ul>
                                    <p class="media-comment">
                                        {{ comment.comment }}
                                    </p>
                                    <a class="btn btn-info btn-circle text-uppercase" data-toggle="collapse" :href="'#replyOne-' + index" v-on:click="openComment(index)"><span class="glyphicon glyphicon-share-alt"></span> Reply</a>
                                    <a class="btn btn-warning btn-circle text-uppercase" data-toggle="collapse" :href="'#replyOne-' + index" v-on:click="openComment(index)"><span class="glyphicon glyphicon-comment"></span> {{ Object.keys(comment.replies).length }} comments</a>
                                </div>
                            </div>
                            <div class="collapse" :id="'replyOne-' + index">
                                <ul class="media-list">
                                    <li class="media media-replied" v-for="(replies, index2) in comment.replies">
                                        <a class="pull-left" href="#">
                                            <img :src="replies.us_avatar" class="gravatar2">
                                        </a>
                                        <div class="media-body">
                                            <div class="well well-lg">
                                                <h4 class="media-heading text-uppercase reviews"><span class="glyphicon glyphicon-share-alt"></span> {{ replies.name }} </h4>
                                                <ul class="media-date reviews list-inline">
                                                    <li class="dd">replied {{ replies.time_ago }}</li>
                                                </ul>
                                                <p class="media-comment">
                                                    {{ replies.comment }}
                                                </p>
                                                <a class="btn btn-info btn-circle text-uppercase" id="reply" v-on:click="openComment(index)"><span class="glyphicon glyphicon-share-alt"></span> Reply</a>
                                            </div>
                                        </div>
                                    </li>
                                </ul>
                            </div>
                            <div class="row" v-if="commentBoxs[index]">
                                <ul class="media-list">
                                    <li class="media media-replied">
                                        <a class="pull-left" href="#">
                                            <img :src="user.user_avatar" class="gravatar2">
                                        </a>
                                        <div class="media-body">
                                            <textarea class="input" placeholder="Add comment..." required v-model="messageList[index]" rows="5" style="width:98%; margin-bottom:10px"></textarea>
                                        </div>
                                        <div class="form-group">
                                            <button class="btn btn-success btn-circle text-uppercase" type="submit" style="float:right; margin-right: 20px;" v-on:click="replyComment(comment.commentid,index)"><span class="glyphicon glyphicon-send"></span>Add Comment</button>
                                        </div>
                                    </li>
                                </ul>
                            </div>
                        </li>
                    </ul>
                </div>
            </div>
        </div>
    </div>
</template>
```

Với template này, chúng ta sẽ có 1 form để tạo comment, sẽ có 1 list các comment, và có nút reply comment. 

Tiếp theo là script để chương trình có thể chạy được, thêm đoạn code sau vào dưới block template vừa được tạo:

```
<script>
    export default {
        props: ['id', 'prj'],
        data: function () {
            return {
                createUrl: laroute.route('comments.save'),
                user: user,
                errorComment: null,
                errorReply: null,
                message: null,
                messageList: [],
                taskId: this.id,
                projectId: this.prj,
                commentsData: [],
                comments: 0,
                commentBoxs: [],
                commentreplies: [],
                replyCommentBoxs: [],
            }
        },
        methods: {
            saveComment: function() {
                if (this.message != null && this.message != ' ') {
                    this.errorComment = null;
                    axios.post(this.createUrl, {
                        task_id: this.taskId,
                        comment_text: this.message,
                        member_id: this.user.id,
                        project_id: this.projectId,
                    }).then(res => {
                        if (res.data.success) {
                            this.commentsData.push({ "commentid": res.data.comment_id, "name": this.user.username, "comment": this.message, "time_ago": res.data.time_ago, "us_avatar": this.user.user_avatar, "votes": 0, "reply": 0, "replies": [] });
                            this.message = null;
                        }
                    });
                } else {
                    this.errorComment = "Please enter a comment to save";
                }
            },
            fetchComments() {
                axios.get(laroute.route('comments.index', {taskId: this.taskId}))
                .then(res => {
                    this.commentsData = res.data;
                    if (res.data.length > 0) {
                        this.comments = 1;
                    }
                });
            },
            openComment(index) {
                if (this.user) {
                    if (this.commentBoxs[index]) {
                        this.$set(this.commentBoxs, index, 0);
                    } else {
                        this.$set(this.commentBoxs, index, 1);
                    }
                }
            },
            replyComment(commentId, index) {
                if (this.messageList[index] != null && this.messageList[index] != ' ') {
                    this.errorReply = null;
                    axios.post(this.createUrl, {
                        task_id: this.taskId,
                        comment_text: this.messageList[index],
                        member_id: this.user.id,
                        reply_id: commentId,
                        project_id: this.projectId,
                    }).then(res => {
                        if (res.data.success) {
                            if (!this.commentsData[index].reply) {
                                this.commentsData[index].replies.push({ "commentid": res.data.comment_id, "name": this.user.username, "comment": this.messageList[index], "votes": 0, "time_ago": res.data.time_ago, "us_avatar": this.user.user_avatar });
                                this.commentsData[index].reply = 1;
                            } else {
                                this.commentsData[index].replies.push({ "commentid": res.data.commentId, "name": this.user.name, "comment": this.messageList[index], "votes": 0,  "time_ago": res.data.time_ago, "us_avatar": this.user.user_avatar });
                            }
                            this.messageList[index] = null;
                        }
                    });
                } else {
                    this.errorReply = "Please enter a comment to save";
                }
            },
            replyCommentBox(index) {
                if (this.user) {
                    if (this.replyCommentBoxs[index]) {
                        this.$set(this.replyCommentBoxs, index, 0);
                    } else {
                        this.$set(this.replyCommentBoxs, index, 1);
                    }
                }
            },
        },
        mounted() {
           this.fetchComments();
        }
    }
</script>
```

Nhìn qua đoạn script này một chút, chúng ta đã định nghĩa các methods sau:

- saveComment(): trong hàm này chúng ta sẽ tiến hành gọi method post lên CommentController cụ thể là function store (được định nghĩa ở bên trên). Tiến hành truyền lên các tham số cần thiết để lưu thông tin vào database. Kiểm tra kết quả trả về, nếu lưu vào database thành công thì tiến hành thêm data vào mảng commentsData. ở trong template chúng ta đã tiến hành hiển thì tất cá các comment có trong mảng này ra thành list. Vì vậy khi bạn thêm mới data vào mảng này, thì trên list sẽ tự động hiển thị comment đó lên cho chúng ta.

- fetchComments():  đây là hàm dùng để lấy tất cả comment và thêm vào mảng commentsData. Thực hiện một request get lên Controoler (tới function index) rồi trả về tất cả commnet thu được.

- replyComment(commentId, index)(): Hàm này sẽ giúp chúng ta reply cho một comment, tương tự như hàm saveComment() chỉ khác một điều rằng hàm này có truyền thêm cả commentId lên để lưu vào database, với commentId được truyền lên chúng ta có thể biết được người dùng đang tiến hành reply vào comment nào.

Cần để ý răng, có một thuộc tính được định nghĩa cuối cùng của thẻ script là 

```
	mounted() {
           this.fetchComments();
        }
```

Với việc định nghĩa này, giúp cho hệ thống của bạn load tất cả comment hiển thị ra cho người dùng ngay khi người dùng truy cập vào hệ thống.

