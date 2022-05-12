Приложение GitHub
Создадим консольное приложение, которое в качестве аргумента командной строки будет принимать имя пользователя GitHub и выводить список репозиториев этого пользователя.

Для этого воспользуемся GitHub API. По запросу https://api.github.com/users/USERNAME/repos возвращается список репозиториев пользователя, USERNAME которого указан в ссылке.

Начнём работу с создания нового Node.js приложения.
Создадим папку github-app, откроем её в VS Code, в терминале выполним команду
npm init -y
Создадим два файла:

app.js - основной файл приложения
github.js - в нём пишем логику взаимодействия с API.
Начнём с файла app.js
Импортируем в него модуль github. Так как это самостоятельно созданный модуль, в качестве параметра метода require() указываем путь к файлу с кодом модуля
const github = require('./github');
Создадим переменную username в которой сохраним аргумент командной строки, переданный при запуске программы

const username = process.argv[2];
Используем импортированный объект github, у которого есть свойство getRepos() - функция, возвращающая список репозиториев пользователя.
Параметры функции getRepos() - username - имя пользователя и функция обратного вызова, принимающая два параметра - error - ошибка и repos - полученные данные, в нашем случае - список репозиториев.
В теле функции обработаем ошибку и выведем в консоль названия репозиториев

github.getRepos(username, (error, repos) => {
  if(error) return console.error(error.message);

  repos.forEach(repo => console.log(repo.name));
});
Переходим к файлу github.js. Наше приложение будет общаться с сервером по протоколу https. Для этого в Node.js есть встроенный модуль https, аналогичный модулю http.
Для отправки запроса к API используем метод get() , который позволяет получить данные от сервера.
Импортируем модуль https и напишем код функции getRepos(). Параметр функции - username - имя пользователя GitHub.
У метода https.get() два параметра: URL, по которому отправляется запрос, и функция обратного вызова, принимающая один параметр - ответ сервера responce, сокращенно res.
Свойство res.statusCode возвращает ответ сервера. Ответ 200 свидетельствует об успешном подключении, любой другой говорит о проблеме с подключением.
Экспортируем модуль github как объект, со свойством getRepos и значением getRepos:
  const https = require('https');
  function getRepos(username) {
    https.get(`https://api.github.com/users/${username}/repos`, res => {
      console.log(res.statusCode);
    })
  }
  module.exports = {getRepos}
Запустим файл app. Аргументом командной строки при запуске файла укажем любой известный нам репозиторий:

node app goldbergyoni
Приложение возвращает ошибку 403.
Код ответа на статус ошибки "HTTP 403 Forbidden" указывает, что сервер понял запрос, но отказывается его авторизовать.
При этом, если мы попробуем запросить те же данные из бразера, перейдя по ссылке https://api.github.com/users/goldbergyoni/repos , получим страницу с нужными нам данными.

Для запроса к API необходимо указать заголовок User-Agent. Браузер при переходе по ссылке добавляет этот заголовок сам, в приложении Node.js его необходимо указать.

Перед отправкой запроса создадим объект option, который отправим вместе с запросом

const option = {
  hostname: 'api.github.com',
  path: `/users/${username}/repos`,
  headers: {
    'User-Agent': 'github-app'
  }
}
Первым параметром метода https.get() укажем не URL-адрес, по которому отправляется запрос, а option.
Приложение возвращает статус 200. который сообщает об успешном подключении.

Обработка входящих данных.
Практически всё в Node.js, в том числе общение с сервером, реализуется асинхронно, при помощи событий и потоков. Информация от сервера приходит по частям.
У ответа сервера response (res) есть событие data, которое сработает тогда, когда от сервера придёт часть запрошенной информации. Подпишемся на это событие и выведем полученные данные в консоль
function getRepos(username) {
  const option = {
    hostname: 'api.github.com',
    path: `/users/${username}/repos`,
    headers: {
      'User-Agent': 'github-app'
    }
  }
  https.get(option, res => {
    res.on('data', data => console.log(data));
  })
};
В консоли несколько объектов Buffer. Преобразовать данные в текст можно или используя метод data.toString(), или воспользовавшись методом res.setEncoding('utf-8').
Чтобы объединить фрагменты данных, создадим переменную body = '' и все фрагменты данных будем к ней присоединять

function getRepos(username) {
  const option = {
    hostname: 'api.github.com',
    path: `/users/${username}/repos`,
    headers: {
      'User-Agent': 'github-app'
    }
  }
  https.get(option, res => {
    res.setEncoding('utf-8');
    let body = '';
    res.on('data', data => body += data);
  })
};
У метода response (res) есть событие end, которое сработает когда передача данных закончится.
При наступлении этого события при помощи метода JSON.parse(body) преобразуем полученные данные в массив

function getRepos(username) {
  let result;
  const option = {
    hostname: 'api.github.com',
    path: `/users/${username}/repos`,
    headers: {
      'User-Agent': 'github-app'
    }
  }
  https.get(option, res => {
    res.setEncoding('utf-8');
    let body = '';
    res.on('data', data => body += data);
    res.on('end', () => {
      result = JSON.parse(body);
    });
  })
};
Как передать полученные данные в модуль app, который импортирует модуль github. Обратите внимание, что модуль app ожидает функцию getRepos() с двумя параметрами - username, и функцией обратного вызова у которой тоже два параметра (error, repos). Эту функцию обратного вызова необходимо указать параметром функции getRepos() в модуле github. Назовём её done - стандартное название подобных функций. И внутри метода response (res) при наступлении события end вызовем функцию done(null, result);. Первый параметр функции - null - отсутствие ошибки, второй параметр - result - массив с репозиториями.
function getRepos(username, done) {
  const option = {
    hostname: 'api.github.com',
    path: `/users/${username}/repos`,
    headers: {
      'User-Agent': 'github-app'
    }
  }
  https.get(option, res => {
    res.setEncoding('utf-8');
    let body = '';
    res.on('data', data => body += data);
    res.on('end', () => {
      const result = JSON.parse(body);
      done(null, result);
    });
  })
};
Обработка ошибок При работе приложения ошибки могут быть в следующих случаях
приложение запускается без имени пользователя
ошибка при отправке запроса, если указано несуществующее имя пользователя
ошибка при получении ответа от сервера
при преобразовании полученных данных в массив Ошибки обрабатываем в модуле github и передаём их в модуль app указывая первым параметром функции done()
чтобы обработать ошибку, когда приложение запускается без имени пользователя, проверяем, пришёл ли в функцию done() параметр username
if(!username) return done(new Error('Необходимо указать имя пользователя'));
Ошибка запроса - событие метода request error.
Создадим переменную request указав её значением метод https.get()
req.on('error', error => done(new Error('Не удалось отправить запрос')));
об ошибке при получении ответа от сервера свидетельствует статус ответа отличный от 200
if(res.statusCode !== 200) return done(new Error('Ошибка при работе с сервером'));
Для обработки ошибок в синхронных методах, к которым относится JSON.parse() используется конструкция try {} catch() {}
try {
  const result = JSON.parse(body);
  done(null, result);
} catch(error) {
  done(new Error('Не удалось обработать данные'));
}
Код функции с обработанными ошибками

function getRepos(username, done) {
  if (!username) return done(new Error('Необходимо указать имя пользователя'));
  const option = {
    hostname: 'api.github.com',
    path: `/users/${username}/repos`,
    headers: {
      'User-Agent': 'github-app',
    },
  };
  const req = https.get(option, (res) => {
    res.setEncoding('utf-8');
    if (res.statusCode === 200) {
      let body = '';
      res.on('data', (data) => (body += data));
      res.on('end', () => {
        try {
          const result = JSON.parse(body);
          done(null, result);
        } catch (error) {
          done(new Error('Не удалось обработать данные'));
        }
      });
    } else {
      done(new Error(`Ошибка при работе с сервером ${res.statusCode} ${res.statusMessage}`));
    }
  });
  req.on('error', (error) => done(new Error('Не удалось отправить запрос')));
}