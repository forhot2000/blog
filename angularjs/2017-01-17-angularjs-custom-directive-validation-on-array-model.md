---
title: AngularJS Custom Directive Validation on Array Model
categories: angular
author: forhot2000@qq.com
date: 2017/01/17
---

我们常常需要自定义一些输入控件，如选择用户控件，在 angularjs 中允许我们添加自定义 directive，于是，
接下来我们就用 directive来实现选择用户控件。

```js
app.directive('userSelect', function($q, $window) {
  function asyncOpenUserSelectDialog(users, opts) {
    var deferred = $q.defer();
    // here is a demo, replace it with your code
    $window.setTimeout(function() {
      deferred.resolve([{ name: 'bob' }, { name: 'john' }]);
    }, 100);
    return deferred.promise;
  }

  return {
    restrict: 'EAC',
    scope: {
      users: '='
    },
    template: '<span>{% raw %}{{names}}{% endraw %}</span> ' +
              '<button ng-click="open()">select</button> ' +
              '<button ng-click="clear()">clear</button>',
    link: function(scope, elem, attrs) {
      function getNames(users) {
        if (!users) {
          return null;
        }
        return users.map(function(user) { return user.name; }).join(', ');
      }

      scope.open = function() {
        asyncOpenUserSelectDialog(scope.users).then(function(selectedUsers) {
          scope.users = selectedUsers;
        });
      };

      scope.clear = function() {
        scope.users = [];
      };

      scope.$watch('users', function(users) {
        scope.names = getNames(users);
      });
    }
  };
});
```

当 directive 添加好了之后，我们在 html 中使用这个 directive 了，非常方便，并且代码简洁。

```html
<user-select users="users"></user-select>
or
<span user-select users="users"></span>
or
<div user-select users="users"></div>
```

然后，我们需要给选择用户控件添加一个多选的开关。

在 directive 的 scope 属性上添加 `multi: '='`，这样我们可以在 scope 中读取到 multi 的值，于是对
directive 进行如下修改。

```js
app.directive('userSelect', function($q, $window) {
  function asyncOpenUserSelectDialog(users, opts) {
    var deferred = $q.defer();
    // here is a demo, replace it with your code
    $window.setTimeout(function() {
      var users = [{ name: 'bob' }, { name: 'john' }];
      deferred.resolve(opts.multi ? users : users[0]);
    }, 100);
    return deferred.promise;
  }

  return {
    restrict: 'EAC',
    scope: {
      users: '=',
      multi: '='
    },
    template: '<span>{% raw %}{{names}}{% endraw %}</span> ' +
              '<button ng-click="open()">select</button> ' +
              '<button ng-click="clear()">clear</button>',
    link: function(scope, elem, attrs) {
      function getNames(users) {
        if (!users) {
          return null;
        }
        return users.map(function(user) { return user.name; }).join(', ');
      }

      function getName(user) {
        if (!user) {
          return null;
        }
        return user.name;
      }

      scope.open = function() {
        asyncOpenUserSelectDialog(scope.users, { multi: scope.multi }).then(function(selectedUsers) {
          scope.users = selectedUsers;
        });
      };

      scope.clear = function() {
        scope.users = scope.multi ? [] : null;
      };

      scope.$watch('users', function(users) {
        scope.names = scope.multi ? getNames(users) : getName(users);
      });
    }
  };
});
```

然后，在 html element 上添加 `multi="true/false"` 属性。

```html
<user-select users="users" multi="true"></user-select>
or
<span user-select users="users" multi="true"></span>
or
<div user-select users="users" multi="true"></div>
```

请注意：因为实际使用中我们不需要在运行中改变 mulit 的值，所以在我们的代码中没有监测 multi 值的变化，请根据需要添加监控该属性。

```js
      scope.$watch('multi', function(value) {
        scope.clear();
      });
```

接下来，我们要把选择用户控件放到 form 中，并验证所选用户不能为空。

Angularjs 提供了 form 的 validate 功能，`ng-required` 用来验证 model 不为空，默认在 NgModelController 上提供了
`$isEmpty` 方法检查 model value 不是 undefined, '', null，但是我们的 model 可能是 array，所以我们需要覆盖默认的
$isEmpty 方法。

首先，我们需要使用 NgModelController，则需要在 directive 中添加 `require: 'ngModel'` 属性，然后 link 方法中的第
四个参数就会是 NgModelController 了，如 `link: function(scope, elem, attrs, ctrl)` 中的 ctrl 即 NgModelController。

同时，我们不能在使用 `users` 属性，而需要使用 `ng-model` 属性为 directive 设置 model。

所以，我们再次修改我们的 directive 代码。

```js
app.directive('userSelect', function($q, $window) {
  function asyncOpenUserSelectDialog(users, opts) {
    var deferred = $q.defer();
    // here is a demo, replace it with your code
    $window.setTimeout(function() {
      var users = [{ name: 'bob' }, { name: 'john' }];
      deferred.resolve(opts.multi ? users : users[0]);
    }, 100);
    return deferred.promise;
  }

  return {
    restrict: 'EAC',
    scope: {
      ngModel: '=',
      multi: '='
    },
    require: 'ngModel',
    template: '<span>{% raw %}{{names}}{% endraw %}</span> ' +
              '<button ng-click="open()">select</button> ' +
              '<button ng-click="clear()">clear</button>',
    link: function(scope, elem, attrs, ctrl) {
      function getNames(users) {
        if (!users) {
          return null;
        }
        return users.map(function(user) { return user.name; }).join(', ');
      }

      function getName(user) {
        if (!user) {
          return null;
        }
        return user.name;
      }

      scope.open = function() {
        asyncOpenUserSelectDialog(scope.users, { multi: scope.multi }).then(function(selectedUsers) {
          ctrl.$setViewValue(selectedUsers);
        });
      };

      scope.clear = function() {
        ctrl.$setViewValue(scope.multi ? [] : null);
      };

      scope.$parent.$watch(attrs.ngModel, function(users) {
        scope.users = users;
      });

      scope.$watch('users', function(users) {
        scope.names = scope.multi ? getNames(users) : getName(users);
      });

      ctrl.$isEmpty = function(value) {
        return !value || (scope.multi && value.length === 0);
      };
    }
  };
});
```

最后，在 html 中将选择用户控件放入到 form 中，并添加 name 属性和错误提示。

```html
<form name="editForm">
  <div>
    <label>users:</label>
    <span user-select name="users" ng-model="users" multi="true" ng-required="true" class="form-input"></span>
    <span class="error" ng-show="editForm.users.$error.required">speaker is required!</span>
  </div>
</form>
```

完整代码示例：

<iframe width="100%" height="450" src="https://jsfiddle.net/forhot2000/ny0e5w02/embedded/" allowfullscreen="allowfullscreen" frameborder="0"></iframe>
