# Адекватная разработка на yii2
## Вступление
Последний год, из-за большого количества проектов и постоянной нехватки рабочих рук и трезвых умов, приходится часто искать и общаться с разными веб-разработчиками. При этом, в личной беседе в мессенджере, приходится не меньше часа или двух рассказывать о том, как у меня устроена работа, какие методы в разработке я приветствую, а какие нет. Постоянно показываю примеры своего кода и даю ссылки на youtube-видео интересных докладов, в которых раскрываются принципы, которым я привык следовать (или, хотя бы, стремиться).

Собственно говоря, поэтому я и решил описать в небольшой статье, то, что приходится проговаривать каждый раз с каждым программистом, которого я хочу подключить к работе.

Принципы, которые я тут описываю, не претендуют на истину в последней инстанции. Многие подходы уже описаны не раз, некоторые подсмотрены в многочисленных докладах на тему yii2. В итоге, за несколько лет очень активной разработки большого количества разных проектов на этом фреймворке - я сформировал своего рода “лучшие практики”, которые лежат в основе всех моих проектов последнего времени. Ну а более старые проекты - подтягиваются по мере наличия свободного времени.

На текущий момент у меня в продакшене и на поддержке находятся больше 15-ти проектов на yii2, ряд из которых достаточно сложные (содержат более 100 моделей со сложной бизнес-логикой, всеми видами тестов). Плюс к этому, в разработке все время 1-3 новых проекта и парочка из старых на переделке (в связи с новыми бизнес-требованиями).  В таком плотном режиме такие слова как переиспользование кода, разделение ответственности, тестирование - уже далеко не просто звуки. 

## Контроль версий, тестирование и деплой
Почти все проекты хостятся на гитхабе в приватных репозиториях. При необходимости подключить к работе новых программистов - им выдается доступ на конкретный репозиторий. При этом, в настройках репозитория пуш в мастер запрещен всем, кроме меня. То есть - все через пул-реквест. Если коллега оказывается ответственным и адекватным - позже ему тоже разрешается пуш в мастер-ветку.

То, что попадает в мастер-ветку - либо улетает в продакшн - либо, при наличии тестов, запускает тестирование в Travis (сейчас github запустил свой аналог трависа и пайплайнов битбакета - actions. Я его пока не щупал).

На продакшене у меня обычно настроено автоматический запуск миграций. Но композер автоматические не обновляю. Если это необходимо, я выполняю composer install -vv руками, если вижу, что .lock файл изменился (а его я обновляю и храню в репозитории)

## Подходы к разработке и архитектуре

Основные принципы, которые я стараюсь соблюдать в своих проектах - это возможность переиспользовать код повторно (в других частях проекта, в других проектах) и разделение ответственности. Кроме того, считаю необходимым максимально использовать те плюшки, которые дает фреймворк (расширение классов ActiveQuery для всех AR-моделей, использование DI-контейнера и пр).

### Тонкие модели и контроллеры
Мне кажется  правильным путем содержать модели и контроллеры худыми,
а всю бизнес-логику выносить в отдельные, независимые классы, 
которые можно повторно использоваться и из фронта, и из 
бекенд-приложения и из консоли. Их так же легко протестировать в относительной изоляции.
То есть, слой “модель” из MVC-подхода представлена сразу набором классов:


#### Класс ActiveRecord
Этот класс из коробки дает нам очень много возможностей и есть большой соблазн ими воспользоваться.
Для меня идеальным ActiveRecord классом явяется тот, в котором есть только:

- Правила валидации для различных сценариев работы с моделью
- Связи с другими моделями 
- Лейблы для полей
- Behaviorы, без которых нельзя обойтись (например, которые расширяют модель для работы с файлами, или LinkerBehavior для удобной работы со множественными связями);

То есть никакой лишней бизнес-логики, которую нам хочется иногда засунусть в `BeforeSave()` и `AfterValidate()`, и другие места, нам туда засывавыть не стоит.
Просто хотя бы по тому, чтобы собрать ее в другом месте. В ActiveRecord классе у нас должно быть то, что отвечает или помогает отвечать за запись модели.

То есть, в идеале, в нашей AR моделе необходимо переопределить только следующие методы:
- tableName()
- rules()
- attributeLabels()
- behaviors()
- find()

И добавить различных связей, при этом не забывая использовать inverseOf():
```php
/**
 * @return ParamQuery
 */
public function getParams()
{
    return $this->hasMany(Params::class, ['id' => 'param_id'])->inverseOf('categories');
}
```

И, так как мы активно используем `ActiveQuery`, то чаще избегаем `andWhere()` и и больше создаем своих методов:

```php
/**
 * @return ParamQuery
 */
public function getParamsActive()
{
    return $this->getParams()->active();
}
```


#### Класс ActiveQuery 
 В котором мы прописываем все возможные варианты запросов, которые могут нам понадобится. Обычно это что-то типа:
- active() - для поиска только активных объектов
- byTitle(string $title)  - для поиска по названию
- dropDownData() - для вывода результатов запроса в виде массива для виджетов форм
- и тому подобные конструкции, которые позволяют избегать в коде копипасты различных условий Model::find()->andWhere()...

Вот пример простого варианта ActiveQuery
```php
class CategoryQuery extends \yii\db\ActiveQuery
{


    /** Возвращаем только те, что есть на складе
     * @return CategoryQuery
     */
    public function available()
    {
        return $this->andWhere(['!=', 'available', 0]);
    }

    /** Возвращаем только активные
     * @return CategoryQuery
     */
    public function active()
    {
        return $this->andWhere(['status' => Status::ACTIVE]);
    }

    /** Быстрая выборка для в виде массива для дропдауна
     * @return array
     */
    public function dropDownData()
    {
        return $this
            ->select('title')
            ->indexBy('id')
            ->orderBy('title')
            ->column();
    }

    /**
     * {@inheritdoc}
     * @return Category[]
     */
    public function all($db = null)
    {
        return parent::all($db);
    }

    /**
     * {@inheritdoc}
     * @return Category|null
     */
    public function one($db = null)
    {
        return parent::one($db);
    }
}


```

#### Классы Enum  
Для хранения различных простых вариантов статусов, типов данных и прочих. Плюсом тут является то, что BaseEnum уже из коробки работает с интернационализацией;

#### Классы-фильтры yii\base\Model 
Их может быть много, и для фронта и для бекенда - разные. 
Могут быть и одинаковые, где ветвление на основе прав доступа
происходит внутри. Обычно эти классы я не наследую от AR модели, 
а наследую от base/Model, отдельно перечисляя все поля-признаки, 
по которым необходимо делать поиск;

#### Классов бизнес-логики 
Для них я обычно определяю отдельный неймспес типа app\logic или common\logic\ и далее складываю их вместе, или группирую по моделям с которыми они работают. Эти классы обычно не наследуют ничего, имеют два публичных метода - конструктор и метод execute() или run(), который и выполняет основное действие класса. Остальные методы в таких классах обычно приватные. Все действия с моделями, которые требуют каких-то действий, помимо сохранения самой AR, описываются в этих классах. То есть, в самих AR классах никаких BeforeSave(), AfterSave() - стараюсь не использовать, все это прописывается через события в классах логики. У такого подхода есть ряд плюсов:
 
- вся бизнес-логика в одном месте
- она отлично покрывается тестами (чего не сделаешь, если она размазана по контроллерам или экшенам)
- у нас тонкие модели
- у нас тонкие контроллеры
- классы бизнес логики мы можем вызвать из любого места нашего приложения (веб, консоль)
- мы можем преконфигурировать эти классы использую DI-контейнер

Один из существующих подходов к yii2-разработке призывает в классах ActiveRecord не описывать rules() и attrubuteLabels(), а наследоваться от этого класса создавая один или несколько классов-форм, для их последующего использования при генерации форм и сохранении объектов. Мне лично этот подход не нравится. Создание отдельных классов-форм я считаю правильным делом, только если сама форма является композитной и должна работать с полями различных AR-моделей. Во всех остальных случаях я предпочитаю использовать такой замечательный механизм как сценарии в классах AR. В реальной жизни редко бывает большое количество сценариев работы с данными конкретной модели (обычно их можно разделить на “редактирование пользователем”, “редактирование админом”, “создание” - для этих случаев разделить правила валидации - это задача пары строк, и нет необходимости плодить лишние абстракции).  
