---
title: "Laravel Tips"
date: 2019-05-02T14:15:06+07:00
tags: [php, laravel]
---

## Order by Mutator

    function getFullNameAttribute() {
        return $this->attributes['first_name'] . ' ' . $this->attributes['last_name'];
    }

    $clients = Client::get()->sortBy('full_name'); // works!
## Default ordering in global scope

    protected static function boot()
    {
        parent::boot();
        // Order by name ASC
        static::addGlobalScope('order', function (Builder $builder) {
            $builder->orderBy('name', 'asc');
        });
    }
## truy vấn điều kiện where

    # where name AND email
    \App\User::whereNameAndEmail('phanlyhuynh','lyhuynh@gmail.com')->first();
    # where name OR email
    \App\User::whereNameOrEmail('huynh','huynh@gmail.com')->get();

## giá trị default cho relation

    public function user()
    {
        return $this->belongsTo('App\User')->withDefault(function ($user) {
            $user->name = 'Guest Author';
        });
    }

## cache dữ liệu
    $value = Cache::remember('users', $minutes, function () {
        return DB::table('users')->get();
    });
    $value = Cache::rememberForever('users', function () {
        return DB::table('users')->get();
    });
## Sử dụng fresh() để truy vấn database và lấy phiên bản mới của item hiện tại

    $user = \App\User::first();
    $user->name = "Something new";
    $user = $user->fresh(); // chú ý rằng nó sẽ trả về giá trị mới, nó không ảnh hưởng tới model hiện tại
    dump($user->name); // nó sẽ trả về tên gốc, gốc phải "Something newm"

## muốn rehydrate model đã tồn tại, chúng ta sử dụng refresh()

    $flight = App\Flight::where('number', 'FR 999')->first();
    $flight->number = 'FR 111';
    $flight->refresh();
    $flight->number; // "FR 999"
##  chuyển hướng 301

    Route::redirect('/here', '/there', 301);
## truy vấn vào model và lấy cả relation của nó

    $postComments = Post::with('comments)->get();
    $postComments = Post::with('comments')->has('comments')->get();
## Increments and Decrements thuộc tính của model

    $article = Article::find($article_id);
    $article->read_count++;
    $article->save();

    #OR
    $article = Article::find($article_id);
    $article->increment('read_count');

    #OR
    Article::find($article_id)->increment('read_count');
    Article::find($article_id)->increment('read_count', 10); // +10
    Product::find($produce_id)->decrement('stock'); // -1

## truy vấn với điều kiện ngày tháng năm

    User::whereDate('created_at', date('Y-m-d'));
    User::whereDay('created_at', date('d'));
    User::whereMonth('created_at', date('m'));
    User::whereYear('created_at', date('Y'));

## Order by relationship

    # first
    public function latestPost()
    {
        return $this->hasOne(\App\Post::class)->latest();
    }
    #seconds
    $users = Topic::with('latestPost')->get()->sortByDesc('latestPost.created_at');
## hạn chế sử dụng if else trong truy vấn

    $query = Author::query();
    $query->when(request('filter_by') == 'likes', function ($q) {
        return $q->where('likes', '>', request('likes_amount', 0));
    });
    $query->when(request('filter_by') == 'date', function ($q) {
        return $q->orderBy('created_at', request('ordering_rule', 'desc'));
    });
## điều kiện where, orwhere

    $q->where(function ($query) {
        $query->where('gender', 'Male')
            ->where('age', '>=', 18);
    })->orWhere(function($query) {
        $query->where('gender', 'Female')
            ->where('age', '>=', 65);
    });