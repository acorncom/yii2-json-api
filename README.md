
Implementation of JSON API specification for the Yii framework
==================================================================
[![Latest Stable Version](https://poser.pugx.org/tuyakhov/yii2-json-api/v/stable.png)](https://packagist.org/packages/tuyakhov/yii2-json-api)
[![Scrutinizer Code Quality](https://scrutinizer-ci.com/g/tuyakhov/yii2-json-api/badges/quality-score.png?b=master)](https://scrutinizer-ci.com/g/tuyakhov/yii2-json-api/?branch=master) [![Build Status](https://scrutinizer-ci.com/g/tuyakhov/yii2-json-api/badges/build.png?b=master)](https://scrutinizer-ci.com/g/tuyakhov/yii2-json-api/build-status/master)
[![Total Downloads](https://poser.pugx.org/tuyakhov/yii2-json-api/downloads.png)](https://packagist.org/packages/tuyakhov/yii2-json-api)

Installation
------------

The preferred way to install this extension is through [composer](http://getcomposer.org/download/).

Either run

```
php composer.phar require --prefer-dist tuyakhov/yii2-json-api "*"
```

or add

```
"tuyakhov/yii2-json-api": "*"
```

to the require section of your `composer.json` file.

Data Serializing and Content Negotiation:
-------------------------------------------
Controller:
```php
class Controller extends \yii\rest\Controller
{
    public $serializer = 'tuyakhov\jsonapi\Serializer';
    
    public function behaviors()
    {
        return ArrayHelper::merge(parent::behaviors(), [
            'contentNegotiator' => [
                'class' => ContentNegotiator::className(),
                'formats' => [
                    'application/vnd.api+json' => Response::FORMAT_JSON,
                ],
            ]
        ]);
    }
}
```
By default, the value of `type` is automatically pluralized. 
You can change this behavior by setting `tuyakhov\jsonapi\Serializer::$pluralize` property:
```php
class Controller extends \yii\rest\Controller
{
    public $serializer = [
        'class' => 'tuyakhov\jsonapi\Serializer',
        'pluralize' => false,  // makes {"type": "user"}, instead of {"type": "users"}
    ];
}
```
Defining models:
1) Let's define `User` model and declare an `articles` relation 
```php
use tuyakhov\jsonapi\ResourceTrait;
use tuyakhov\jsonapi\ResourceInterface;

class User extends ActiveRecord implements ResourceInterface
{
    use ResourceTrait;
    
    public function getArticles()
    {
        return $this->hasMany(Article::className(), ['author_id' => 'id']);
    }
}
```
2) Now we need to define `Article` model
```php
use tuyakhov\jsonapi\ResourceTrait;
use tuyakhov\jsonapi\ResourceInterface;

class Article extends ActiveRecord implements ResourceInterface
{
    use ResourceTrait;
}
```
3) As the result `User` model will be serialized into the proper json api resource object:
```javascript
{
  "data": {
    "type": "users",
    "id": "1",
    "attributes": {
      // ... this user's attributes
    },
    "relationships": {
      "articles": {
        // ... this user's articles
      }
    }
  }
}
```
Controlling JSON API output
------------------------------
The JSON response is generated by the `tuyakhov\jsonapi\JsonApiResponseFormatter` class which will
use the `yii\helpers\Json` helper internally. This formatter can be configured with different options like
for example the `$prettyPrint` option, which is useful on development for
better readable responses, or `$encodeOptions` to control the output
of the JSON encoding.

The formatter can be configured in the `yii\web\Response::formatters` property of the `response` application
component in the application configuration like the following:

```php
'response' => [
    // ...
    'formatters' => [
        \yii\web\Response::FORMAT_JSON => [
            'class' => 'tuyakhov\jsonapi\JsonApiResponseFormatter',
            'prettyPrint' => YII_DEBUG, // use "pretty" output in debug mode
            'encodeOptions' => JSON_UNESCAPED_SLASHES | JSON_UNESCAPED_UNICODE,
        ],
    ],
],
```
Links
---------------------------
Your resource classes may support HATEOAS by implementing the `LinksInterface`.
The interface contains `getLinks()` method which should return a list of links.
Typically, you should return at least the `self` link representing the URL to the resource object itself.
In order to appear the links in relationships `getLinks()` method should return `self` link. 
Based on this link each relationship will generate `self` and `related` links. 
By default it happens by appending a relationship name at the end of the `self` link of the primary model, 
you can simply change that behavior by overwriting `getRelationshipLinks()` method. 
For example,
```php
class User extends ActiveRecord implements ResourceInterface, LinksInterface
{
    use ResourceTrait;
    
    public function getLinks()
    {
        return [
            Link::REL_SELF => Url::to(['user/view', 'id' => $this->id], true),
        ];
    }
}
```
As the result:
```javascript
{
  "data": {
    "type": "users",
    "id": "1",
    // ... this user's attributes
    "relationships": {
      "articles": {
        // ... article's data
        "links": {
            "self": {"href": "http://yourdomain.com/users/1/relationships/articles"},
            "related": {"href": "http://yourdomain.com/users/1/articles"}
        }
      }
    }
    "links": {
        "self": {"href": "http://yourdomain.com/users/1"}
    }
  }
}
```
Enabling JSON API Input
---------------------------
To let the API accept input data in JSON API format, configure the [[yii\web\Request::$parsers|parsers]] property of the request application component to use the [[tuyakhov\jsonapi\JsonApiParser]] for JSON input
```php
'request' => [
  'parsers' => [
      'application/vnd.api+json' => 'tuyakhov\jsonapi\JsonApiParser',
  ]
]
```
By default it parses a HTTP request body so that you can populate model attributes with user inputs.
For example the request body:
```javascript
{
  "data": {
    "type": "users",
    "id": "1",
    "attributes": {
        "first-name": "Bob",
        "last-name": "Homster"
    }
  }
}
```
Will be resolved into the following array:
```php
// var_dump($_POST);
[
    "User" => [
        "first_name" => "Bob", 
        "last_name" => "Homster"
    ]
]
```
So you can access request body by calling `\Yii::$app->request->post()` and simply populate the model with input data:
```php
$model = new User();
$model->load(\Yii::$app->request->post());
```
By default type `users` will be converted into `User` (singular, camelCase) which corresponds to the model's `formName()` method (which you may override).
You can override the `JsonApiParser::formNameCallback` property which refers to a callback that converts 'type' member to form name.
Also you could change the default behavior for conversion of member names to variable names ('first-name' converts into 'first_name') by setting `JsonApiParser::memberNameCallback` property.

Examples
--------
Controller:
```php
class UserController extends \yii\rest\Controller
{
    public $serializer = 'tuyakhov\jsonapi\Serializer';

    /**
     * @inheritdoc
     */
    public function behaviors()
    {
        return ArrayHelper::merge(parent::behaviors(), [
            'contentNegotiator' => [
                'class' => ContentNegotiator::className(),
                'formats' => [
                    'application/vnd.api+json' => Response::FORMAT_JSON,
                ],
            ]
        ]);
    }
    
    /**
     * @inheritdoc
     */
    public function actions()
    {
        return [
            'create' => [
                'class' => 'tuyakhov\jsonapi\actions\CreateAction',
                'modelClass' => ExampleModel::className()
            ],
            'update' => [
                'class' => 'tuyakhov\jsonapi\actions\UpdateAction',
                'modelClass' => ExampleModel::className()
            ],
            'view' => [
                'class' => 'tuyakhov\jsonapi\actions\ViewAction',
                'modelClass' => ExampleModel::className(),
            ],
            'delete' => [
                'class' => 'tuyakhov\jsonapi\actions\DeleteAction',
                'modelClass' => ExampleModel::className(),
            ],
            'view-related' => [
                'class' => 'tuyakhov\jsonapi\actions\ViewRelatedAction',
                'modelClass' => ExampleModel::className()
            ],
            'update-relationship' => [
                'class' => 'tuyakhov\jsonapi\actions\UpdateRelationshipAction',
                'modelClass' => ExampleModel::className()
            ],
            'delete-relationship' => [
                'class' => 'tuyakhov\jsonapi\actions\DeleteRelationshipAction',
                'modelClass' => ExampleModel::className()
            ],
            'options' => [
                'class' => 'yii\rest\OptionsAction',
            ],
        ];
    }
}

```

Model:
```php
class User extends ActiveRecord implements LinksInterface, ResourceInterface
{
    use ResourceTrait;
    
    public function getLinks()
    {
        $reflect = new \ReflectionClass($this);
        $controller = Inflector::camel2id($reflect->getShortName());
        return [
            Link::REL_SELF => Url::to(["$controller/view", 'id' => $this->getId()], true)
        ];
    }
}
```

Configuration file `config/main.php`:
```php
return [
    // ...
    'components' => [
        'request' => [
            'parsers' => [
                'application/vnd.api+json' => 'tuyakhov\jsonapi\JsonApiParser',
            ]
        ],
        'response' => [
            'format' => \yii\web\Response::FORMAT_JSON,
            'formatters' => [
                \yii\web\Response::FORMAT_JSON => 'tuyakhov\jsonapi\JsonApiResponseFormatter'
            ]
        ],
        'urlManager' => [
            'rules' => [
                [
                    'class' => 'yii\rest\UrlRule',
                    'controller' => 'user',
                    'extraPatterns' => [
                        'GET {id}/<name:\w+>' => 'view-related',
                        'PATCH {id}/relationships/<name:\w+>' => 'update-relationship',
                        'DELETE {id}/relationships/<name:\w+>' => 'delete-relationship',
                        '{id}/<name:\w+>' => 'options'
                    ],
                    'except' => ['index'],
                ],

            ]
        ]
        // ...
    ]
    // ...
]
```