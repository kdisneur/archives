---
layout: post
title: Unit test an AngularJS app without any AngularJS dependencies
tags:
  - angularjs
  - commonjs
  - javascript
  - unit-testing
---
The main problem I have with `AngularJS` applications is to unit test them. The framework comes with a good dependency
injection system allowing us to mock every dependency. But we have to load the library in our unit test in order to initialize
our controllers, services,...

A basic `AngularJS` application where the developer cares about the global namespace pollution could look like this:

```javascript
// my_app.js
angular.module('MyApp', ['localstorage', 'mm.foundation', 'ngResource', 'satellizer', 'ui.router']);

// my_app/authentication_ctrl.js
(function() {
  var AuthenticationCtrl = function($scope, $state, AuthenticationService) {
    // code omitted
  };

  angular.module('MyApp').controller('AuthenticationCtrl', ['$scope', '$state', 'AuthenticationService', AuthenticationCtrl]);
})();

// my_app/authentication_service.js
(function() {
  var AuthenticationService = function($auth, $q, $rootScope) {
    // code omitted
  };

  angular.module('MyApp').service('AuthenticationService', ['$auth', '$q', '$rootScope', AuthenticationService]);
})();
```

Here we have an application where `AngularJS` is in the global namespace but everyelse is scoped in anonymous function to not
pollute the global namespace. Seems good!

## Unit testing problem

The problem with the previous implementation comes when we try to unit test our app. If we want to test our
`authentication_service`, we would like to require this file only. Doing that we will have an error because `MyApp`
is not defined yet and `AngularJS` is not included. Outch!

We have to load the whole framework to run our unit test... It's no longer unit...

We could find lot of sources explaining how to unit test our app this way (even the [AngularJS documentation](https://docs.angularjs.org/guide/unit-testing))

Here a copy-paste from the official `AngularJS` documentation:

```javascript
describe('PasswordController', function() {
  beforeEach(module('app'));

  var $controller;

  beforeEach(inject(function(_$controller_){
    // The injector unwraps the underscores (_) from around the parameter names when matching
    $controller = _$controller_;
  }));

  describe('$scope.grade', function() {
    it('sets the strength to "strong" if the password length is >8 chars', function() {
      var $scope = {};
      var controller = $controller('PasswordController', { $scope: $scope });
      $scope.password = 'longerthaneightchars';
      $scope.grade();
      expect($scope.strength).toEqual('strong');
    });
  });
});
```

`AngularJS`, are you serious? We need to use `module`, `inject`,...? We also have to use your `$controller` service to
initialize our controller?

No, I am not ready to **unit** test my classes like this.

## Enter in the CommonJS world

`CommonJS` mainly adds two interesting concepts:

* `module`: we can scope our code in a module and export just some part of our code;
* `require`: a nice method that allows us to require a module where we want.

If we look at our first try, we could rewrite it like this:

```javascript
// my_app.js
angular.module('MyApp', ['localstorage', 'mm.foundation', 'ngResource', 'satellizer', 'ui.router'])
       .controller('AuthenticationCtrl', ['$scope', '$state', 'AuthenticationService', require('my_app/authentication_ctrl')])
       .service('AuthenticationService', ['$auth', '$q', '$rootScope', require('my_app/authentication_service')]);

// my_app/authentication_ctrl.js
require.register('my_app/authentication_ctrl', function(exports, require, module) {
  var AuthenticationCtrl = function($scope, $state, AuthenticationService) {
    // code omitted
  };

  module.exports = AuthenticationCtrl;
});

// my_app/authentication_service.js
require.register('my_app/authentication_service', function(exports, require, module) {
  var AuthenticationService = function($auth, $q, $rootScope) {
    // code omitted
  };

  module.exports = AuthenticationService;
});
```

Neat, right? We have only one place where we define our application. Moreover, our controllers and services don't need
to depend to `AngularJS` anymore.

Our **unit** tests can now looks like this:

```javascript
describe('AuthenticationCtrl', function() {
  var promise        = require('test_helpers/promise');
  var describedClass = require('deploy_my_codes/controllers/authentication_controller');
  var authenticationSpy, dashboardSpy, fakeAuthenticationService, fakeScope, fakeState, logoutSpy, signInSpy, subject;

  beforeEach(function() {
    authenticationSpy = sinon.spy();
    dashboardSpy      = sinon.spy();
    fakeAuthenticationService = {};
    fakeScope = {};
    fakeState = {};
    logoutSpy = sinon.spy();
    signInSpy = sinon.spy();
    subject   = describedClass(fakeScope, fakeState, fakeAuthenticationService);
  });

  beforeEach(function() {
    fakeAuthenticationService.authenticate = function(provider) {
      var deferred = promise.defer();
      if (provider === 'github') {
        authenticationSpy();
        deferred.resolve({});
      } else {
        deferred.reject({});
      }

      return deferred.promise;
    };

    fakeAuthenticationService.logout = function(provider) {
      var deferred = promise.defer();
      logoutSpy();
      deferred.resolve({});
      return deferred.promise;
    };

    fakeState.go = function(state) {
      switch (state) {
        case 'dashboard':
          dashboardSpy();
          break;
        case 'sign_in':
          signInSpy();
          break;
      }
    };
  });

  describe('when action #authenticate is called', function() {
    it('redirects to the dashboard', function() {
      fakeScope.authenticate('github');
      sinon.assert.calledOnce(authenticationSpy);
      sinon.assert.calledOnce(dashboardSpy);
    });
  });

  describe('when action #authenticate is called', function() {
    it('redirects to the sign in', function() {
      fakeScope.logout();
      sinon.assert.calledOnce(logoutSpy);
      sinon.assert.calledOnce(signInSpy);
    });
  });
});
```

Tada! We have a real unit testing file now.

As we can see we require a custom promise helper. Why? Because we need a dummy and synchronous implementation of promise
without any dependences to `AngularJS`.

## Don't drink too much Kool-Aid

The main drawback with `CommonJS` is its support. As we can see in `my_app.js` file, we don't require `AngularJS`...
We're using it as a global variable (beurk). It's because `AngularJS` doesn't work this way.

You can take a look to [browserify](http://browserify.org/) if you want to go further.

Personnally I prefer to stick to `CommonJS` and force me to not use global variables. If I need them I create a module
and use it everywhere.

For example, for `Underscore.js`:

```
require.register('underscore', function(exports, require, module) {
  module.exports = window._;
});
```

And when I need underscore in one of my class, I just have to require it:

```javascript
var _ = require('underscore');
```
