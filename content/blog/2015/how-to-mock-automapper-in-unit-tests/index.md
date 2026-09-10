---
title: "How to isolate AutoMapper in Unit Tests?"
date: 2015-05-20T00:00:00+00:00
publishdate: 2015-05-20T00:00:00+00:00
lastmod: 2015-05-20T00:00:00+00:00
tags: ["Unit Testing", "Library"]
comments: true
draft: false
disableBreadcrumb: true
---

<p><strong>Introduction</strong></p>
<p>When it comes to Unit testing, the rule is to isolate&nbsp;all the dependencies and focus only on the unit of code you are testing. I have been silently ignoring this for <a href="http://automapper.org/">AutoMapper</a> until I <!-- more -->found a proper way to isolate and replace the behavior of mapping.</p>
<p>When I was looking for a proper way to do this, I come across many posts but none of them were having full working example without any additional layer between the function/class that wants to use AutoMapper and AutoMapper itself.</p>
<p><strong>IMappingEngine</strong></p>
<p>AutoMapper comes with a static class Mapper which has almost all the methods we need to work with AutoMapper. I we want to create a new mapping, we can use Mapper.CreateMap() method (we can also use AutoMapper <a href="https://github.com/AutoMapper/AutoMapper/wiki/Configuration">Profile instances</a>) and if we want to perform mapping between pre-configured objects we can use Mapper.Map() method.</p>
<p>The Mapper class exposes a property Engine which holds all the mapping configurations. This property is of type IMappingEngine.</p>
<p>When we invoke Map method on Mapper class, it in-turn calls Map method of Engine property.</p>
<p>So, all we need to do to isolate and control the behavior of mapping in our unit test is to replace the Engine property with our own implementation.</p>
<p><strong>But, how?</strong></p>
<p>The Engine is static property in Mapper class which is also static. We know very well that it is nearly impossible to replace the implementation of static members without using some fancy tools or test patterns.</p>
<p>Fortunately, the type of Engine property is interface IMappingEngine. This means, we can declare IMappingEngine as a dependency of SUT and ask DI framework to provide it with the instance, in this case Mapper.Engine.</p>
<p>Here is how it looks like if you use Ninject</p>

```cs
kernel.Bind<IMappingEngine>().ToConstant(Mapper.Engine);
```

<p>And with ASP.NET 5 built-in DI</p>

```cs
services.AddInstance<IMappingEngine>(Mapper.Engine);
```

<p>In the unit test method, you need to create Test Double which implements IMappingEngine and create SUT with Test Double as input for dependency. The easiest way to create Test Double is to create it as a Mock using your favourite mocking framework.</p>
<p>Here is an example using Moq,</p>

```cs
var mockMapper = new Mock<IMappingEngine>();

mockMapper.Setup(mock => mock.Map<TargetType>(It.IsAny<SourceType>())).Returns(targetInstance);
```

<p><strong>Full example</strong></p>

```cs
public class User
    {
        public int Id { get; set; }
        public string FirstName { get; set; }
        public string LastName { get; set; }
        public int Age { get; set; }
    }

    public class UserDto
    {
        public int Id { get; set; }
        public string FirstName { get; set; }

    }

    public class Sut
    {
        private readonly IMappingEngine _mapper;

        public Sut(IMappingEngine mapper)
        {
            _mapper = mapper;
        }

        public User Run(UserDto dto)
        {
            return _mapper.Map<User>(dto);
        }
    }

    public class SutTests
    {
        [Fact]
        public void RunShouldReturnUserCorrectly()
        {
            // arrange
            var stubUser = new User();
            var stubUserDto = new UserDto();
            var mockMapper = new Mock<IMappingEngine>();
            mockMapper.Setup(mock => mock.Map<User>(It.Is<UserDto>(stubUserDto))).Returns(stubUser);

            var target = new Sut(mockMapper.Object);

            // act
            var actual = target.Run(stubUserDto);

            // assert
            Assert.AreEquals(stubUser, actual);

        }
    }
```
