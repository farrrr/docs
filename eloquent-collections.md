# Eloquent：集合

- [簡介](#introduction)
- [可用的方法](#available-methods)
- [自訂集合](#custom-collections)

<a name="introduction"></a>
## 簡介

由 Eloquent 回傳的所有多種結果的集合都是 `Illuminate\Database\Eloquent\Collection` 物件的實例，包含經由 `get` 方法取得或是經由關聯來存取的結果。Eloquent 集合物件繼承了 Laravel [基底集合](/docs/{{version}}/collections)，所以它自然繼承了許多可用於流暢地與 Eloquent 模型的底層陣列合作的方法。

當然，所有的集合也都可以作為迭代器，讓你像是操作簡單的 PHP 陣列一般去遍歷集合：

    $users = App\User::where('active', 1)->get();

    foreach ($users as $user) {
        echo $user->name;
    }

然而，集合比陣列更強大並提供 map 或 reduce 等各種鏈結操作的直觀介面。例如，讓我們來移除所有未啟用的模型並收集其餘每個使用者的名字：

    $users = App\User::where('active', 1)->get();

    $names = $users->reject(function ($user) {
        return $user->active === false;
    })
    ->map(function ($user) {
        return $user->name;
    });

> **注意：** 雖然大多數 Eloquent 集合方法回傳一個新的 Eloquent 集合的實例，但是 `pluck`、`keys`、`zip`、`collapse`、`flatten` 和 `flip` 方法是回傳一個[基底集合](/docs/{{version}}/collections)ˇ的實例。

<a name="available-methods"></a>
## 可用的方法

### 基底集合

所有 Eloquent 集合繼承了基底 [Laravel 集合](/docs/{{version}}/collections)物件；因此，他們繼承了所有基底集合類別所提供的強大的方法：

<style>
    #collection-method-list > p {
        column-count: 3; -moz-column-count: 3; -webkit-column-count: 3;
        column-gap: 2em; -moz-column-gap: 2em; -webkit-column-gap: 2em;
    }

    #collection-method-list a {
        display: block;
    }
</style>

<div id="collection-method-list" markdown="1">
[all](/docs/{{version}}/collections#method-all)
[chunk](/docs/{{version}}/collections#method-chunk)
[collapse](/docs/{{version}}/collections#method-collapse)
[contains](/docs/{{version}}/collections#method-contains)
[count](/docs/{{version}}/collections#method-count)
[diff](/docs/{{version}}/collections#method-diff)
[each](/docs/{{version}}/collections#method-each)
[every](/docs/{{version}}/collections#method-every)
[filter](/docs/{{version}}/collections#method-filter)
[first](/docs/{{version}}/collections#method-first)
[flatten](/docs/{{version}}/collections#method-flatten)
[flip](/docs/{{version}}/collections#method-flip)
[forget](/docs/{{version}}/collections#method-forget)
[forPage](/docs/{{version}}/collections#method-forpage)
[get](/docs/{{version}}/collections#method-get)
[groupBy](/docs/{{version}}/collections#method-groupby)
[has](/docs/{{version}}/collections#method-has)
[implode](/docs/{{version}}/collections#method-implode)
[intersect](/docs/{{version}}/collections#method-intersect)
[isEmpty](/docs/{{version}}/collections#method-isempty)
[keyBy](/docs/{{version}}/collections#method-keyby)
[keys](/docs/{{version}}/collections#method-keys)
[last](/docs/{{version}}/collections#method-last)
[map](/docs/{{version}}/collections#method-map)
[merge](/docs/{{version}}/collections#method-merge)
[pluck](/docs/{{version}}/collections#method-pluck)
[pop](/docs/{{version}}/collections#method-pop)
[prepend](/docs/{{version}}/collections#method-prepend)
[pull](/docs/{{version}}/collections#method-pull)
[push](/docs/{{version}}/collections#method-push)
[put](/docs/{{version}}/collections#method-put)
[random](/docs/{{version}}/collections#method-random)
[reduce](/docs/{{version}}/collections#method-reduce)
[reject](/docs/{{version}}/collections#method-reject)
[reverse](/docs/{{version}}/collections#method-reverse)
[search](/docs/{{version}}/collections#method-search)
[shift](/docs/{{version}}/collections#method-shift)
[shuffle](/docs/{{version}}/collections#method-shuffle)
[slice](/docs/{{version}}/collections#method-slice)
[sort](/docs/{{version}}/collections#method-sort)
[sortBy](/docs/{{version}}/collections#method-sortby)
[sortByDesc](/docs/{{version}}/collections#method-sortbydesc)
[splice](/docs/{{version}}/collections#method-splice)
[sum](/docs/{{version}}/collections#method-sum)
[take](/docs/{{version}}/collections#method-take)
[toArray](/docs/{{version}}/collections#method-toarray)
[toJson](/docs/{{version}}/collections#method-tojson)
[transform](/docs/{{version}}/collections#method-transform)
[unique](/docs/{{version}}/collections#method-unique)
[values](/docs/{{version}}/collections#method-values)
[where](/docs/{{version}}/collections#method-where)
[whereLoose](/docs/{{version}}/collections#method-whereloose)
[zip](/docs/{{version}}/collections#method-zip)
</div>

<a name="custom-collections"></a>
## 自訂集合

如果你需要使用一個自訂的 `Collection` 物件與你自己的擴充方法，你可以在模型中覆寫 `newCollection` 方法：

    <?php

    namespace App;

    use App\CustomCollection;
    use Illuminate\Database\Eloquent\Model;

    class User extends Model
    {
        /**
         * 建立一個新的 Eloquent 集合實例。
         *
         * @param  array  $models
         * @return \Illuminate\Database\Eloquent\Collection
         */
        public function newCollection(array $models = [])
        {
            return new CustomCollection($models);
        }
    }

一旦你定義了 `newCollection` 方法，在任何 Eloquent 回傳該模型的 `Collection` 實例的時候，你將會接收到一個你的自訂集合的實例。如果你想要在你的應用程式每個模型中使用自訂的集合，你應該在所有的模型繼承的基底模型中覆寫 `newCollection` 方法。
