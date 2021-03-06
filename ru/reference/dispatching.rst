Диспетчер контроллеров
======================
Компонент :doc:`Phalcon\\Mvc\\Dispatcher <../api/Phalcon_Mvc_Dispatcher>`  отвечает за инициализацию контроллеров и выполнения в них действий, для MVC
приложения. Понимание его работы и его возможностей помогает нам получить больше возможностей предоставляемых фреймворком.

Цикл работы диспетчера
----------------------
Это важнейший процесс, который имеет много общего с работой MVC, особенно в части работы контроллеров. Работа контроллера вызывается диспетчером.
Файлы контроллера считываются, загружаются, инициализируются, чтобы затем выполнить необходимые действия. Если действие направляет поток на другой
котроллер/действие (action), диспетчер контроллера стартует снова. Для лучшей иллюстрации в примере ниже показан приблизительный процесс происходящий
внутри :doc:`Phalcon\\Mvc\\Dispatcher <../api/Phalcon_Mvc_Dispatcher>`:

.. code-block:: php

    <?php

    // Цикл диспетчера
    while (!$finished) {

        $finished = true;

        $controllerClass = $controllerName . "Controller";

        // Создание экземпляра класса контроллера, с помощью автозагрузчика
        $controller = new $controllerClass();

        // Выполнение действия
        call_user_func_array(array($controller, $actionName . "Action"), $params);

        // Значение переменной должно быть изменено при необходимости запуска другого контроллера
        $finished = true;

    }

Этот код, безусловно, нуждается в дополнительных проверках и доработке, но здесь наглядно показана типичная последовательность операций в диспетчере.

События при работе диспетчера
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
:doc:`Phalcon\\Mvc\\Dispatcher <../api/Phalcon_Mvc_Dispatcher>` может отправлять события :doc:`EventsManager <events>` если это необходимо.
События вызываются с помощью типа "dispatch". Некоторые события, при возвращении false, могут остановить активную операцию.
Поддерживаются следующие события:

+----------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------+-----------------------+
| Название события     | Время срабатывания                                                                                                                                                                                           | Прерывает операцию? | Triggered to          |
+======================+==============================================================================================================================================================================================================+=====================+=======================+
| beforeDispatchLoop   | До запуска цикла диспетчера. В этот момент диспетчер не знает, существуют ли контроллеры или действия, которые должны быть выполнены. Диспетчер владеет только информацией поступившей из маршрутизатора     | Да                  | Listeners             |
+----------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------+-----------------------+
| beforeDispatch       | До выполнения цикла диспетчера. В этот момент диспетчер не знает, существуют ли контроллеры или действия, которые должны быть выполнены. Диспетчер знает только информацию, поступившую из маршрутизатора    | Да                  | Listeners             |
+----------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------+-----------------------+
| beforeExecuteRoute   | До выполнения действия в контроллере. В этой точке контроллер инициализирован и знает о существовании действия (action)                                                                                      | Да                  | Listeners/Controllers |
+----------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------+-----------------------+
| afterExecuteRoute    | После выполнения действия в контроллере. Не останавливает текущую операцию, используйте это событие только для завершения/очистки после выполненного действия                                                | Нет                 | Controllers           |
+----------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------+-----------------------+
| beforeNotFoundAction | Когда действие не найдено в контроллере                                                                                                                                                                      | Нет                 | Listeners/Controllers |
+----------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------+-----------------------+
| beforeException      | До вызова диспетчером любого исключения                                                                                                                                                                      | Да                  | Listeners             |
+----------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------+-----------------------+
| afterDispatch        | После выполнения цикла диспетчера. Не останавливает текущую операцию, используйте это событие только для завершения/очистки после выполненного действия                                                      | Да                  | Listeners             |
+----------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------+-----------------------+
| afterDispatchLoop    | После завершения цикла диспетчера                                                                                                                                                                            | Нет                 | Listeners             |
+----------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------+-----------------------+

В обучающем материале :doc:`INVO <tutorial-invo>` показано, как воспользоваться диспетчером событий для реализации фильтра безопасности :doc:`Acl <acl>`.

В примере ниже показано как прикрепить слушателей (listeners) к событиям контроллера:

.. code-block:: php

    <?php

    use Phalcon\Mvc\Dispatcher as MvcDispatcher,
        Phalcon\Events\Manager as EventsManager;

    $di->set('dispatcher', function(){

        // Создание менеджера событий
        $eventsManager = new EventsManager();

        // Прикрепление функции-слушателя для событий типа "dispatch"
        $eventsManager->attach("dispatch", function($event, $dispatcher) {
            //...
        });

        $dispatcher = new MvcDispatcher();

        // Связывание менеджера событий с диспетчером
        $dispatcher->setEventsManager($eventsManager);

        return $dispatcher;

    }, true);

Экземпляр контроллера автоматически выступает в качестве слушателя для событий, так что вы можете реализовать методы в самом контроллере:

.. code-block:: php

    <?php

    class PostsController extends \Phalcon\Mvc\Controller
    {

        public function beforeExecuteRoute($dispatcher)
        {
            // Выполняется перед каждым найденным действием
        }

        public function afterExecuteRoute($dispatcher)
        {
            // Выполняется после каждого выполненного действия
        }

    }

Переадресация на другое действие
--------------------------------
Цикл диспетчера позволяет перенаправить поток на другой контроллер/действие. Это очень полезно, для проверки может ли пользователь иметь
доступ к определенным функциям, перенаправления пользователя на другую страницу или просто для повторного использования кода.

.. code-block:: php

    <?php

    class PostsController extends \Phalcon\Mvc\Controller
    {

        public function indexAction()
        {

        }

        public function saveAction($year, $postTitle)
        {

            // .. сохраняем данные и перенаправляем пользователя

            // Перенаправляем на действие index
            $this->dispatcher->forward(array(
                "controller" => "post",
                "action" => "index"
            ));
        }

    }

Имейте ввиду, использование метода "forward" - это не то же самое что редирект в HTTP. Хотя внешне результат будет таким же.
Метод "forward" не перезагружает текущую страницу, все перенаправления выполняются в одном запросе, тогда как HTTP редирект требует два
запроса для завершения процесса.

Пример перенаправлений:

.. code-block:: php

    <?php

    // Направляем поток на другое действие текущего контроллера
    $this->dispatcher->forward(array(
        "action" => "search"
    ));

    // Направляем поток на другое действие текущего контроллера с передачей параметров
    $this->dispatcher->forward(array(
        "action" => "search",
        "params" => array(1, 2, 3)
    ));

Метод forward принимает следующие параметры:

+----------------+--------------------------------------------------------+
| Параметр       | Описание                                               |
+================+========================================================+
| controller     | Правильное имя контроллера для вызова                  |
+----------------+--------------------------------------------------------+
| action         | Правильное название действия для вызова                |
+----------------+--------------------------------------------------------+
| params         | Массив параметров для действия (action)                |
+----------------+--------------------------------------------------------+
| namespace      | Пространство имён, которому принадлежит контроллер     |
+----------------+--------------------------------------------------------+

Получение параметров
--------------------
Если текущий маршрут содержит именованные параметры, вы можете получить их в контроллере, представлении или любом другом компоненте,
расширяющим :doc:`Phalcon\\DI\\Injectable <../api/Phalcon_DI_Injectable>`.

.. code-block:: php

    <?php

    class PostsController extends \Phalcon\Mvc\Controller
    {

        public function indexAction()
        {

        }

        public function saveAction()
        {

            // Получение параметра title, находящимся в параметрах URL
            $title = $this->dispatcher->getParam("title");

            // Получение параметра year, пришедшего из URL и отфильтрованного как число
            $year = $this->dispatcher->getParam("year", "int");
        }

    }

Обработка исключений "Не найдено"
---------------------------------
Используйте возможности :doc:`EventsManager <events>` для установки событий, выполняемых при отсутствии требуемого контроллера/действия.

.. code-block:: php

    <?php

    use Phalcon\Dispatcher,
        Phalcon\Mvc\Dispatcher as MvcDispatcher,
        Phalcon\Events\Manager as EventsManager;

    $di->set('dispatcher', function() {

        // Создаем менеджер событий 
        $eventsManager = new EventsManager();

        // Прикрепляем слушателя
        $eventsManager->attach("dispatch:beforeException", function($event, $dispatcher, $exception) {

            switch ($exception->getCode()) {
                case Dispatcher::EXCEPTION_HANDLER_NOT_FOUND:
                case Dispatcher::EXCEPTION_ACTION_NOT_FOUND:
                    $dispatcher->forward(array(
                        'controller' => 'index',
                        'action' => 'show404'
                    ));
                    return false;
            }
        });

        $dispatcher = new MvcDispatcher();

        //Прикрепляем менеджер событий к диспетчеру
        $dispatcher->setEventsManager($eventsManager);

        return $dispatcher;

    }, true);

Реализация собственных диспетчеров
----------------------------------
Для создания диспетчеров необходимо реализовать интерфейс :doc:`Phalcon\\Mvc\\DispatcherInterface <../api/Phalcon_Mvc_DispatcherInterface>` и подменить
диспетчер Phalcon.
