# Ресурсный роутинг (часть вторая)

[>>> Первая часть](/rails-routing/resource-routing-the-rails-default-1)

h4. Вложенные ресурсы

Нормально иметь ресурсы, которые логически подчинены другим ресурсам. Например, предположим ваше приложение включает эти модели:

```ruby
class Magazine < ActiveRecord::Base
  has_many :ads
end

class Ad < ActiveRecord::Base
  belongs_to :magazine
end
```

Вложенные маршруты позволяют перехватить эти отношения в вашем роутинге. В этом случае можете включить такое объявление маршрута:

```ruby
resources :magazines do
  resources :ads
end
```

В дополнение к маршрутам для magazines, это объявление также создаст маршруты для ads в `AdsController`. URL с ad требует magazine:

| Метод HTTP | Путь                                 | Экшн    | Использование                                                                         |
| ---------- | ------------------------------------ | ------- | ------------------------------------------------------------------------------------- |
| GET        | /magazines/:magazine_id/ads          | index   | отображает список всей рекламы для определенного журнала                              |
| GET        | /magazines/:magazine_id/ads/new      | new     | возвращает форму HTML для создания новой рекламы, принадлежащей определенному журналу |
| POST       | /magazines/:magazine_id/ads          | create  | создает новую рекламу, принадлежащую указанному журналу                               |
| GET        | /magazines/:magazine_id/ads/:id      | show    | отражает определенную рекламу, принадлежащую определенному журналу                    |
| GET        | /magazines/:magazine_id/ads/:id/edit | edit    | возвращает форму HTML для редактирования рекламы, принадлежащей определенному журналу |
| PATCH/PUT  | /magazines/:magazine_id/ads/:id      | update  | обновляет определенную рекламу, принадлежащую определенному журналу                   |
| DELETE     | /magazines/:magazine_id/ads/:id      | destroy | удаляет определенную рекламу, принадлежащую определенному журналу                     |

Также будут созданы маршрутные хелперы, такие как `magazine_ads_url` и `edit_magazine_ad_path`. Эти хелперы принимают экземпляр Magazine как первый параметр (`magazine_ads_url(@magazine)`).

#### Ограничения для вложения

Вы можете вкладывать ресурсы в другие вложенные ресурсы, если хотите. Например:

```ruby
resources :publishers do
  resources :magazines do
    resources :photos
  end
end
```

Глубоко вложенные ресурсы быстро становятся громоздкими. В этом случае, например, приложение будет распознавать URL, такие как

```
/publishers/1/magazines/2/photos/3
```

Соответствующий маршрутный хелпер будет `publisher_magazine_photo_url`, требующий определения объектов на всех трех уровнях. Действительно, эта ситуация достаточно запутана, так что в [статье](http://weblog.jamisbuck.org/2007/2/5/nesting-resources) Jamis Buck предлагает правило хорошей разработки на Rails:

TIP: _Ресурсы никогда не должны быть вложены глубже, чем на 1 уровень._

### Концерны маршрутов

Концерны маршрутов (Routing Concerns) позволяют объявлять обычные маршруты, которые затем могут быть повторно использованы внутри других ресурсов и маршрутов.

```ruby
concern :commentable do
  resources :comments
end

concern :image_attachable do
  resources :images, only: :index
end
```

Эти концерны могут быть использованы в ресурсах, чтобы избежать дублирования кода и разделить поведение между несколькими маршрутами.

```ruby
resources :messages, concerns: :commentable

resources :posts, concerns: [:commentable, :image_attachable]
```

Также их можно использовать в любом месте внутри маршрутов, например в вызове scope или namespace:

```ruby
namespace :posts do
  concerns :commentable
end
```

### Создание путей и URL из объектов

В дополнение к использованию маршрутных хелперов, Rails может также создавать пути и URL из массива параметров. Например, предположим у вас есть этот набор маршрутов:

```ruby
resources :magazines do
  resources :ads
end
```

При использовании magazine_ad_path, можно передать экземпляры `Magazine` и `Ad` вместо числовых ID:

```erb
<%= link_to "Ad details", magazine_ad_path(@magazine, @ad) %>
```

Можно также использовать `url_for` с набором объектов, и Rails автоматически определит, какой маршрут вам нужен:

```erb
<%= link_to "Ad details", url_for([@magazine, @ad]) %>
```

В этом случае Rails увидит, что `@magazine` это `Magazine` и `@ad` это `Ad` и поэтому использует хелпер `magazine_ad_path`. В хелперах, таких как `link_to`, можно определить лишь объект вместо полного вызова `url_for`:

```erb
<%= link_to "Ad details", [@magazine, @ad] %>
```

Если хотите ссылку только на magazine:

```erb
<%= link_to "Magazine details", @magazine %>
```

Для других экшнов следует всего лишь вставить имя экшна как первый элемент массива:

```erb
<%= link_to "Edit Ad", [:edit, @magazine, @ad] %>
```

Это позволит рассматривать экземпляры ваших моделей как URL, что является ключевым преимуществом ресурсного стиля.

### Определение дополнительных экшнов RESTful

Вы не ограничены семью маршрутами, которые создает роутинг RESTful по умолчанию. Если хотите, можете добавить дополнительные маршруты, применяющиеся к коллекции или отдельным элементам коллекции.

#### Добавление маршрутов к элементам

Для добавления маршрута к элементу, добавьте блок `member` в блок ресурса:

```ruby
resources :photos do
  member do
    get 'preview'
  end
end
```

Это распознает `/photos/1/preview` с GET, и направит его в экшн `preview` `PhotosController`. Это также создаст хелперы `preview_photo_url` и `preview_photo_path`.

В блоке маршрутов к элементу каждое имя маршрута определяет метод HTTP, с которым он будет распознан. Тут можно использовать `get`, `patch`, `put`, `post` или `delete`. Если у вас нет нескольких маршрутов к `элементу`, также можно передать `:on` к маршруту, избавившись от блока:

```ruby
resources :photos do
  get 'preview', :on => :member
end
```

#### Добавление маршрутов к коллекции

Чтобы добавить маршрут к коллекции:

```ruby
resources :photos do
  collection do
    get 'search'
  end
end
```

Это позволит Rails распознать URL, такие как `/photos/search` с GET и направить его в экшн `search` `PhotosController`. Это также создаст маршрутные хелперы `search_photos_url` и `search_photos_path`.

Как и с маршрутами к элементу, можно передать `:on` к маршруту:

```ruby
resources :photos do
  get 'search', :on => :collection
end
```

#### Предостережение

Если вдруг вы захотели добавить много дополнительных экшнов в ресурсный маршрут, нужно остановиться и спросить себя, может от вас утаилось присутствие другого ресурса.