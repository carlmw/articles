# Making JavaScript testing in the browser not suck with Sinon.js (Part 2)

This is the continuation of my post on Sinon.js, the first part can be found [testing_javascript_part1](here).

I'm going to discuss the potential of Sinon's mocks, spies, and stubs

## Getting started
Assuming you've read the first part you should already have a test suite up and running with your framework of choice.

## Mocks
Mocks replace APIs with fake methods. We can use mocks to specify "expectations". Expectations allow us to test how our code responds to return values from our mocked API.

### Benefits:
* Tests run more quickly
* Bugs can be traced more easily
* Tests are easier to understand

Heres an example test utilising a mock:
  
    module('TeaBreak');

    test('it should save data', 1, function () {
      var mock = sinon.mock(Network);
      
      mock.expects('save')
        .withArgs({cuppa: 'lovely'});

      new TeaBreak({cuppa: 'lovely'}).enjoy();

      mock.verify();
    });

Here we're setting an expection. We're expecting the "save" method on the "Network" object to be called with the correct data.

Lets run the test and watch it fail.

![Failing test with a mock expectation](failing_mock.png)

Now to write some code to make it pass.

    var TeaBreak = function (data) {
      this.data = data;
    };

    TeaBreak.prototype = {
      enjoy: function () {
        Network.save(this.data);
      }
    };

And run the test.

![Passing test with a mock](passing_mock.png)
