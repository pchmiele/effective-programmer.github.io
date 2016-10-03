+++
author = "Przemysław Chmielewski"
date = "2016-03-16T14:45:28+02:00"
description = "Meteor testowanie end-to-end z wykorzystaniem frameworka mocha oraz chimp"
tags = ["chimp", "end-to-end testing", "javascript", "meteor", "mocha", "testing"]
categories = ["javascript", "meteor"]
menu = ""
banner = "banners/negativespace-15-750x410.jpg"
title = "Meteor testowanie end-to-end"
+++

Meteor jest fantastycznym frameworkiem jeżeli chodzi o czas w jakim jesteśmy w stanie stworzyć w pełni działający serwis. Niestety po dziś dzień nie ma oficjalnego rozwiązania jeżeli chodzi o jego testowanie. Jednak ma się to zmienić w najbliższym czasie wraz z pojawieniem się wersji 1.3 (wciąż in-progress). Powinniśmy wtedy otrzymać informacje o tym jakie są oficjalnie wspierane narzędzia do testowania oraz jakie są wytyczne i praktyki proponowane przez zespół rozwijający projekt. Dodatkowo oficjalne tutoriale i przykładowe aplikacje powinny zostać rozszerzone o sekcje związane z testowaniem. Niestety zanim to nastąpi trzeba sobie radzić samemu. Postanowiłem zaprezentować jedną z dróg, która powinna być zbliżona do tej która zostanie wprowadzona w wersji 1.3. Całość zaprezentuję w oparciu o przykładowy projekt przygotowany przez twórców Meteora.

Zaczynamy od stworzenia przykładowego projektu
```
meteor create --example todos
```
Do testowania potrzebujemy frameworków Chimp oraz Mocha
```bash
npm install -g chimp
npm install -g mocha
```
**Chimp** służy do uruchamiania testów end to end w przeglądarce. Testy mogą być pisane w dowolnym ze wspieranych frameworków. Akutalnie jest to mocha oraz cucumber a trwają prace nad wsparciem dla jasmine. Testy uruchamiane są każdorazowo gdy zmodyfikowany zostanie kod aplikacji i/lub testów. Najprawdopodobniej to właśnie Chimp zostanie wybrany jako oficjalny framework do uruchamiania testów end to end do aplikacji tworzonych w Meteorze.

**Mocha** natomiast raczej nie wymaga opisu. Każdy kto kiedykolwiek pisał testy automatyczne do aplikacji stworzonych w nodejs (javascript) zapewne miał styczność z tym frameworkiem (ewentualnie z jasmine, która jest bardzo podobna jeżeli chodzi o składnie).

## Testy

Tworzymy strukturę katalogów w których trzymane będą testy.
```bash
{meteor_project}\tests - główny katalog z testami
{meteor_project}\tests\mocha - testy pisane we frameworku Mocha
{meteor_project}\tests\mocha\client - testy pisane uruchamiane w przeglądarce\po stronie klienta
{meteor_project}\tests\mocha\client\example-test.js - przykładowy plik z testami o których dalej będzie mowa
```
W pliku z testami (example_test.js) dodajemy:
```javascript
function getTodoLists() {
    return browser.elements('.list-todo').value;
}

describe('Views', function() {
    describe('/', function() {
        beforeEach(function() {
            browser.url('http://localhost:3000');
            browser.waitForExist('.title-page');
        });

        it('page title should be set by the Meteor method @watch', function () {
            expect(browser.getTitle()).to.equal('Todos - All your todos synced wherever you happen to be');

        });

        it('user should be able to create todo list @watch', function () {
            var initialTodoListLength = getTodoLists().length;
            browser.click('.js-new-list');

            var actualTodoListLength = getTodoLists().length;
            var expectedTodoListLength = initialTodoListLength + 1;
            expect(actualTodoListLength).to.equal(expectedTodoListLength);
        });
    });

    describe('/join', function() {
        it('new user should be able to create new account @watch', function () {
            browser.url('http://localhost:3000/join');
            browser.waitForText('.title-auth');

            browser.setValue('[name="email"]', 'asd2@mail.com');
            browser.setValue('[name="password"]', 'password');
            browser.setValue('[name="confirm"]', 'password');

            browser.click('.page.auth .wrapper-auth form .btn-primary');
        });
    });
});
```
Uruchamiamy nasza aplikację (musimy dodatkowo ustawić zmienną środowiskową IS_MIRROR=1 w konsoli lub w używanym przez nas IDE)
```
set IS_MIRROR=1
meteor run
```
Uruchamiamy nasze testy
```
chimp --ddp=http://localhost:3000 --watch --mocha --path=tests
```
## Czyszczenie bazy i dane tesowe

Testy powinny się uruchomić i wszystkie z nich powinny przejść. Niestety przy kolejnym uruchomieniu rezultat będzie inny. Jeden z testów się załamie a wszystko przez to, że wszystkie testy w trakcie działania modyfikują  bazę danych. Musimy sprawić aby, każdy z testów był jak najbardziej niezależny od pozostałych. Do dzieła.

Tworzymy i dodajemy nową paczkę
```
meteor create --package fixtures
meteor add fixtures
```
W nowo powstałym katalogu podmieniamy zawartość pliku
```
{meteor_project}\packages\fixtures\fixture.js
```
na
```
var defaultData  = [
    {name: "Meteor Principles",
        items: ["Data on the Wire",
            "One Language",
            "Database Everywhere",
            "Latency Compensation",
            "Full Stack Reactivity",
            "Embrace the Ecosystem",
            "Simplicity Equals Productivity"
        ]
    },
    {name: "Languages",
        items: ["Lisp",
            "C",
            "C++",
            "Python",
            "Ruby",
            "JavaScript",
            "Scala",
            "Erlang",
            "6502 Assembly"
        ]
    }
];

var timestamp = (new Date()).getTime();

Meteor.methods({
    'fixtures/loadFixtures': function(){
        if (process.env.IS_MIRROR) {
            console.log('Loading default fixtures');
            Accounts.createUser({
                email: 'email@example.com',
                password: '123456'
            });

            _.each(defaultData, function(list) {
                var list_id = Lists.insert({name: list.name,
                    incompleteCount: list.items.length});

                _.each(list.items, function(text) {
                    Todos.insert({listId: list_id,
                        text: text,
                        createdAt: new Date(timestamp)});
                    timestamp += 1; // ensure unique timestamp.
                });
            });

            console.log('Finished loading default fixtures');
        } else {
            console.log('loadFixtures failed');
        }
    },

    'fixtures/clearDB': function(){
        if (process.env.IS_MIRROR) {
            console.log('Clear DB');

            var collectionsRemoved = 0;
            var db = Meteor.users.find()._mongo.db;
            db.collections(function (err, collections) {

                var appCollections = _.reject(collections, function (col) {
                    return col.collectionName.indexOf('velocity') === 0 ||
                        col.collectionName === 'system.indexes';
                });

                _.each(appCollections, function (appCollection) {
                    appCollection.remove(function (e) {
                        if (e) {
                            console.error('Failed removing collection', e);
                            fut.return('fail: ' + e);
                        }
                        collectionsRemoved++;
                        console.log('Removed collection');
                        if (appCollections.length === collectionsRemoved) {
                            console.log('Finished resetting database');
                        }
                    });
                });
            });

            console.log('Finished clearing');
        } else {
            console.log('clearDB failed');
        }
    }
});
```
Dodaliśmy właśnie dwie metody dostępne z kodu testów:
```
fixtures/clearDB - odpowiedzialna za czyszczenie całej bazy danych
fixtures/loadFixtures - umożliwia ładowanie defaultowych danych
```
Dodajemy ich wywołanie do naszych testów
```
describe('Views', function() {
    beforeEach(function() {
        server.call('fixtures/clearDB');
        server.call('fixtures/loadFixtures');
    });
...
```
Teraz nasze testy powinny przechodzić za każdym razem. A rezultat powinien być podobny do poniższego.
![Meteor testowanie z wykorzystaniem chimp i mocha](/gifs/meteor-mocha-testing-resized.gif)

Cały kod jest dostępny na moim [githubie](https://github.com/pchmiele/effective-programmer-blog/tree/master/blog/meteor/testing/testing-with-mocha-example).

Źródła:

Chimp - https://chimp.readme.io/  
Mocha - https://mochajs.org/  
Jasmine - http://jasmine.github.io/