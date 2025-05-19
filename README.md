<script type="text/javascript">
        var gk_isXlsx = false;
        var gk_xlsxFileLookup = {};
        var gk_fileData = {};
        function filledCell(cell) {
          return cell !== '' && cell != null;
        }
        function loadFileData(filename) {
        if (gk_isXlsx && gk_xlsxFileLookup[filename]) {
            try {
                var workbook = XLSX.read(gk_fileData[filename], { type: 'base64' });
                var firstSheetName = workbook.SheetNames[0];
                var worksheet = workbook.Sheets[firstSheetName];

                // Convert sheet to JSON to filter blank rows
                var jsonData = XLSX.utils.sheet_to_json(worksheet, { header: 1, blankrows: false, defval: '' });
                // Filter out blank rows (rows where all cells are empty, null, or undefined)
                var filteredData = jsonData.filter(row => row.some(filledCell));

                // Heuristic to find the header row by ignoring rows with fewer filled cells than the next row
                var headerRowIndex = filteredData.findIndex((row, index) =>
                  row.filter(filledCell).length >= filteredData[index + 1]?.filter(filledCell).length
                );
                // Fallback
                if (headerRowIndex === -1 || headerRowIndex > 25) {
                  headerRowIndex = 0;
                }

                // Convert filtered JSON back to CSV
                var csv = XLSX.utils.aoa_to_sheet(filteredData.slice(headerRowIndex)); // Create a new sheet from filtered array of arrays
                csv = XLSX.utils.sheet_to_csv(csv, { header: 1 });
                return csv;
            } catch (e) {
                console.error(e);
                return "";
            }
        }
        return gk_fileData[filename] || "";
        }
        </script><!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Тесты по сервисному центру</title>
    <script src="https://cdn.tailwindcss.com"></script>
</head>
<body class="bg-gray-100 flex items-center justify-center min-h-screen">
    <div id="app" class="bg-white p-8 rounded-lg shadow-lg w-full max-w-2xl">
        <!-- Экран регистрации -->
        <div id="register-screen" class="space-y-4">
            <h1 class="text-2xl font-bold text-center">Регистрация</h1>
            <input id="username" type="text" placeholder="Введите имя" class="w-full p-2 border rounded">
            <input id="password" type="password" placeholder="Введите пароль" class="w-full p-2 border rounded">
            <button onclick="register()" class="w-full bg-blue-500 text-white p-2 rounded hover:bg-blue-600">Зарегистрироваться</button>
            <p class="text-center">Уже есть аккаунт? <a href="#" onclick="showLogin()" class="text-blue-500">Войти</a></p>
        </div>

        <!-- Экран входа -->
        <div id="login-screen" class="space-y-4 hidden">
            <h1 class="text-2xl font-bold text-center">Вход</h1>
            <input id="login-username" type="text" placeholder="Введите имя" class="w-full p-2 border rounded">
            <input id="login-password" type="password" placeholder="Введите пароль" class="w-full p-2 border rounded">
            <button onclick="login()" class="w-full bg-blue-500 text-white p-2 rounded hover:bg-blue-600">Войти</button>
            <p class="text-center">Нет аккаунта? <a href="#" onclick="showRegister()" class="text-blue-500">Зарегистрироваться</a></p>
        </div>

        <!-- Экран выбора теста -->
        <div id="test-selection-screen" class="space-y-4 hidden">
            <h1 class="text-2xl font-bold text-center">Выберите тест</h1>
            <div id="test-list" class="space-y-2"></div>
            <button onclick="logout()" class="w-full bg-red-500 text-white p-2 rounded hover:bg-red-600">Выйти</button>
        </div>

        <!-- Экран теста -->
        <div id="test-screen" class="space-y-4 hidden">
            <h1 id="test-title" class="text-2xl font-bold text-center"></h1>
            <div id="question-number" class="text-lg font-semibold"></div>
            <div id="question" class="text-lg"></div>
            <div id="options" class="space-y-2"></div>
            <div class="flex justify-between">
                <button onclick="prevQuestion()" class="bg-gray-500 text-white p-2 rounded hover:bg-gray-600">Предыдущий вопрос</button>
                <button onclick="nextQuestion()" class="bg-blue-500 text-white p-2 rounded hover:bg-blue-600">Следующий вопрос</button>
            </div>
            <button onclick="finishTest()" class="w-full bg-green-500 text-white p-2 rounded hover:bg-green-600">Завершить тест</button>
            <button onclick="backToSelection()" class="w-full bg-red-500 text-white p-2 rounded hover:bg-red-600">Вернуться к выбору теста</button>
        </div>

        <!-- Экран результатов -->
        <div id="result-screen" class="space-y-4 hidden">
            <h1 class="text-2xl font-bold text-center">Результаты теста</h1>
            <p id="result-text" class="text-lg"></p>
            <table class="w-full border-collapse">
                <thead>
                    <tr class="bg-gray-200">
                        <th class="border p-2">Вопрос</th>
                        <th class="border p-2">Ваш ответ</th>
                        <th class="border p-2">Правильный ответ</th>
                        <th class="border p-2">Статус</th>
                        <th class="border p-2">Объяснение</th>
                    </tr>
                </thead>
                <tbody id="result-table"></tbody>
            </table>
            <button onclick="restartTest()" class="w-full bg-blue-500 text-white p-2 rounded hover:bg-blue-600">Пройти тест заново</button>
            <button onclick="backToSelection()" class="w-full bg-red-500 text-white p-2 rounded hover:bg-red-600">Вернуться к выбору теста</button>
        </div>
    </div>

    <script>
        const users = JSON.parse(localStorage.getItem('users')) || {};
        let currentUser = null;
        let currentTest = null;
        let currentQuestion = 0;
        let score = 0;
        let userAnswers = [];

        const tests = [
            {
                id: 'test1',
                title: 'Введение в работу сервисного центра',
                questions: [
                    { question: 'Что делают в сервисном центре в первую очередь, чтобы найти проблему устройства?', options: ['Чинят устройство', 'Проводят диагностику', 'Чистят от пыли', 'Обновляют программы'], answer: 1, explanation: 'Диагностика позволяет определить причину неисправности.' },
                    { question: 'Какой процесс помогает выявить проблему, если устройство не включается?', options: ['Проверка кабеля питания', 'Замена процессора', 'Обновление BIOS', 'Чистка кулера'], answer: 0, explanation: 'Проверка кабеля питания — первый шаг при отсутствии включения.' },
                    { question: 'Что делают, если устройство сильно нагревается?', options: ['Обновляют драйверы', 'Проверяют систему охлаждения', 'Заменяют батарею', 'Устанавливают антивирус'], answer: 1, explanation: 'Перегрев часто связан с неисправной системой охлаждения.' },
                    { question: 'Какой процесс улучшает производительность устройства?', options: ['Замена экрана', 'Увеличение оперативной памяти', 'Чистка корпуса', 'Обновление BIOS'], answer: 1, explanation: 'Увеличение оперативной памяти ускоряет работу устройства.' },
                    { question: 'Что проверяют, если устройство не заряжается?', options: ['Процессор', 'Зарядное устройство и порт', 'Жесткий диск', 'Оперативную память'], answer: 1, explanation: 'Неисправность зарядного устройства или порта — частая причина.' },
                    { question: 'Что надевают, чтобы защитить устройство от статического электричества?', options: ['Очки', 'Антистатический браслет', 'Перчатки', 'Маску'], answer: 1, explanation: 'Антистатический браслет предотвращает повреждение компонентов.' },
                    { question: 'Какой предмет защищает от поражения током при работе с устройством?', options: ['Резиновые перчатки', 'Антистатический коврик', 'Очки', 'Маска'], answer: 0, explanation: 'Резиновые перчатки изолируют от электричества.' },
                    { question: 'Что используют для защиты глаз при пайке?', options: ['Антистатический браслет', 'Защитные очки', 'Перчатки', 'Респиратор'], answer: 1, explanation: 'Защитные очки предотвращают повреждение глаз.' },
                    { question: 'Какой предмет предотвращает случайное повреждение платы?', options: ['Отвертка', 'Антистатический коврик', 'Паяльник', 'Мультиметр'], answer: 1, explanation: 'Антистатический коврик защищает плату от статики.' },
                    { question: 'Что делают перед разборкой устройства для безопасности?', options: ['Обновляют ПО', 'Отключают от сети', 'Чистят корпус', 'Проверяют драйверы'], answer: 1, explanation: 'Отключение от сети предотвращает поражение током.' },
                    { question: 'Какой инструмент используют для откручивания винтов в устройствах?', options: ['Мультиметр', 'Отвертка', 'Паяльник', 'Кисточка'], answer: 1, explanation: 'Отвертка нужна для работы с винтами.' },
                    { question: 'Какой прибор измеряет напряжение в цепи?', options: ['Отвертка', 'Мультиметр', 'Паяльник', 'Термометр'], answer: 1, explanation: 'Мультиметр измеряет напряжение.' },
                    { question: 'Что используют для очистки платы от пыли?', options: ['Паяльник', 'Баллон со сжатым воздухом', 'Мультиметр', 'Отвертка'], answer: 1, explanation: 'Сжатый воздух эффективно удаляет пыль.' },
                    { question: 'Какой интерфейс проще в использовании: BIOS или UEFI?', options: ['BIOS, у него текстовый интерфейс', 'UEFI, у него графический интерфейс', 'Оба сложные', 'Оба текстовые'], answer: 1, explanation: 'UEFI имеет графический интерфейс, упрощающий работу.' },
                    { question: 'Какой компонент отвечает за начальную загрузку системы?', options: ['Жесткий диск', 'BIOS/UEFI', 'Оперативная память', 'Процессор'], answer: 1, explanation: 'BIOS/UEFI управляет начальной загрузкой.' },
                    { question: 'Что позволяет UEFI, чего нет в BIOS?', options: ['Поддержка больших дисков', 'Текстовый интерфейс', 'Ограничение загрузки', 'Работа только с 32-бит'], answer: 0, explanation: 'UEFI поддерживает диски больше 2 ТБ.' },
                    { question: 'Какой процесс требуется перед обновлением BIOS/UEFI?', options: ['Форматирование диска', 'Резервное копирование настроек', 'Установка антивируса', 'Чистка кулера'], answer: 1, explanation: 'Резервное копирование защищает настройки.' },
                    { question: 'Какой режим в BIOS/UEFI позволяет снизить энергопотребление?', options: ['Secure Boot', 'Power Saving Mode', 'Legacy Mode', 'Fast Boot'], answer: 1, explanation: 'Power Saving Mode снижает энергопотребление.' },
                    { question: 'Какой ключ чаще всего открывает меню BIOS/UEFI?', options: ['Ctrl', 'Del', 'Shift', 'Alt'], answer: 1, explanation: 'Del чаще всего открывает BIOS/UEFI.' },
                    { question: 'Какой процесс защищает устройство от несанкционированного доступа?', options: ['Установка пароля BIOS', 'Чистка платы', 'Обновление драйверов', 'Замена батареи'], answer: 0, explanation: 'Пароль BIOS ограничивает доступ.' },
                    { question: 'Какой тип диагностики выполняется автоматически при включении?', options: ['POST', 'Антивирусная проверка', 'Проверка драйверов', 'Чистка системы'], answer: 0, explanation: 'POST проверяет оборудование при включении.' },
                    { question: 'Какой инструмент помогает измерить температуру процессора?', options: ['Мультиметр', 'Программное обеспечение', 'Паяльник', 'Отвертка'], answer: 1, explanation: 'Программы вроде AIDA64 измеряют температуру.' },
                    { question: 'Что используют для нанесения термопасты?', options: ['Паяльник', 'Пластиковая лопатка', 'Мультиметр', 'Кисточка'], answer: 1, explanation: 'Пластиковая лопатка равномерно наносит термопасту.' },
                    { question: 'Какой процесс помогает восстановить заводские настройки BIOS/UEFI?', options: ['Сброс CMOS', 'Обновление прошивки', 'Чистка платы', 'Замена процессора'], answer: 0, explanation: 'Сброс CMOS возвращает настройки BIOS/UEFI к заводским.' },
                    { question: 'Какой элемент BIOS/UEFI отвечает за порядок загрузки устройств?', options: ['Boot Priority', 'Secure Boot', 'Power Saving Mode', 'Legacy Mode'], answer: 0, explanation: 'Boot Priority определяет порядок загрузки устройств.' }
                ]
            },
            {
                id: 'test2',
                title: 'Жёсткие диски и SSD: виды, различия и особенности',
                questions: [
                    // Вставьте вопросы из второго теста в том же формате, например:
                    // { question: 'Какой основной компонент HDD отвечает за чтение и запись данных?', options: ['Магнитные пластины', 'Магнитная головка', 'Контроллер', 'Флеш-память'], answer: 1, explanation: 'Магнитная головка считывает и записывает данные на пластины.' },
                    // Продолжите для всех 25 вопросов
                ]
            },
            {
                id: 'test3',
                title: 'Основы диагностики и ремонта ноутбуков Lenovo',
                questions: [
                    // Вставьте вопросы из третьего теста
                ]
            },
            {
                id: 'test4',
                title: 'Основы работы с сетевыми кабелями (теория по обжиму)',
                questions: [
                    // Вставьте вопросы из четвертого теста
                ]
            },
            {
                id: 'test5',
                title: 'Флеш-накопители: устройство, использование и безопасность',
                questions: [
                    // Вставьте вопросы из пятого теста
                ]
            },
            {
                id: 'test6',
                title: 'Сборка ПК: компоненты, этапы и настройка',
                questions: [
                    // Вставьте вопросы из шестого теста
                ]
            },
            {
                id: 'test7',
                title: 'Переустановка Windows и установка ПО',
                questions: [
                    // Вставьте вопросы из седьмого теста
                ]
            },
            {
                id: 'test8',
                title: 'Руководство по iMac',
                questions: [
                    // Вставьте вопросы из восьмого теста
                ]
            },
            {
                id: 'test9',
                title: 'Устройство и принцип работы картриджей',
                questions: [
                    // Вставьте вопросы из девятого теста
                ]
            },
            {
                id: 'test10',
                title: 'Разборка и сборка картриджей',
                questions: [
                    // Вставьте вопросы из десятого теста
                ]
            }
        ];

        function showRegister() {
            document.getElementById('register-screen').classList.remove('hidden');
            document.getElementById('login-screen').classList.add('hidden');
            document.getElementById('test-selection-screen').classList.add('hidden');
            document.getElementById('test-screen').classList.add('hidden');
            document.getElementById('result-screen').classList.add('hidden');
        }

        function showLogin() {
            document.getElementById('register-screen').classList.add('hidden');
            document.getElementById('login-screen').classList.remove('hidden');
            document.getElementById('test-selection-screen').classList.add('hidden');
            document.getElementById('test-screen').classList.add('hidden');
            document.getElementById('result-screen').classList.add('hidden');
        }

        function showTestSelection() {
            document.getElementById('register-screen').classList.add('hidden');
            document.getElementById('login-screen').classList.add('hidden');
            document.getElementById('test-selection-screen').classList.remove('hidden');
            document.getElementById('test-screen').classList.add('hidden');
            document.getElementById('result-screen').classList.add('hidden');
            loadTestList();
        }

        function showTest() {
            document.getElementById('register-screen').classList.add('hidden');
            document.getElementById('login-screen').classList.add('hidden');
            document.getElementById('test-selection-screen').classList.add('hidden');
            document.getElementById('test-screen').classList.remove('hidden');
            document.getElementById('result-screen').classList.add('hidden');
            loadQuestion();
        }

        function showResult() {
            document.getElementById('register-screen').classList.add('hidden');
            document.getElementById('login-screen').classList.add('hidden');
            document.getElementById('test-selection-screen').classList.add('hidden');
            document.getElementById('test-screen').classList.add('hidden');
            document.getElementById('result-screen').classList.remove('hidden');
            displayResults();
        }

        function register() {
            const username = document.getElementById('username').value.trim();
            const password = document.getElementById('password').value.trim();
            if (username && password) {
                if (users[username]) {
                    alert('Пользователь уже существует!');
                } else {
                    users[username] = { password, testResults: {} };
                    localStorage.setItem('users', JSON.stringify(users));
                    alert('Регистрация успешна! Теперь войдите.');
                    showLogin();
                }
            } else {
                alert('Введите имя и пароль!');
            }
        }

        function login() {
            const username = document.getElementById('login-username').value.trim();
            const password = document.getElementById('login-password').value.trim();
            if (users[username] && users[username].password === password) {
                currentUser = username;
                showTestSelection();
            } else {
                alert('Неверное имя или пароль!');
            }
        }

        function loadTestList() {
            const testList = document.getElementById('test-list');
            testList.innerHTML = '';
            tests.forEach(test => {
                const button = document.createElement('button');
                button.className = 'w-full p-2 border rounded hover:bg-gray-200';
                button.innerText = test.title;
                button.onclick = () => startTest(test.id);
                testList.appendChild(button);
            });
        }

        function startTest(testId) {
            currentTest = tests.find(test => test.id === testId);
            currentQuestion = 0;
            score = 0;
            userAnswers = [];
            document.getElementById('test-title').innerText = currentTest.title;
            showTest();
        }

        function loadQuestion() {
            const q = currentTest.questions[currentQuestion];
            document.getElementById('question-number').innerText = `Вопрос ${currentQuestion + 1} из ${currentTest.questions.length}`;
            document.getElementById('question').innerText = q.question;
            const optionsDiv = document.getElementById('options');
            optionsDiv.innerHTML = '';
            q.options.forEach((option, index) => {
                const button = document.createElement('button');
                button.className = 'w-full p-2 border rounded hover:bg-gray-200';
                button.innerText = option;
                button.onclick = () => selectOption(index);
                optionsDiv.appendChild(button);
            });
        }

        function selectOption(index) {
            const q = currentTest.questions[currentQuestion];
            userAnswers[currentQuestion] = { selected: index, correct: index === q.answer };
            if (index === q.answer) score++;
            nextQuestion();
        }

        function nextQuestion() {
            currentQuestion++;
            if (currentQuestion < currentTest.questions.length) {
                loadQuestion();
            } else {
                saveResults();
                showResult();
            }
        }

        function prevQuestion() {
            if (currentQuestion > 0) {
                currentQuestion--;
                loadQuestion();
            }
        }

        function finishTest() {
            saveResults();
            showResult();
        }

        function saveResults() {
            users[currentUser].testResults[currentTest.id] = { score, answers: userAnswers };
            localStorage.setItem('users', JSON.stringify(users));
        }

        function displayResults() {
            document.getElementById('result-text').innerText = `Ваш результат: ${score} из ${currentTest.questions.length}`;
            const resultTable = document.getElementById('result-table');
            resultTable.innerHTML = '';
            currentTest.questions.forEach((q, index) => {
                const row = document.createElement('tr');
                const userAnswer = userAnswers[index] ? q.options[userAnswers[index].selected] : 'Не отвечено';
                const correctAnswer = q.options[q.answer];
                const status = userAnswers[index] && userAnswers[index].correct ? 'Правильно' : 'Неправильно';
                row.innerHTML = `
                    <td class="border p-2">${q.question}</td>
                    <td class="border p-2">${userAnswer}</td>
                    <td class="border p-2">${correctAnswer}</td>
                    <td class="border p-2 ${status === 'Правильно' ? 'text-green-500' : 'text-red-500'}">${status}</td>
                    <td class="border p-2">${q.explanation}</td>
                `;
                resultTable.appendChild(row);
            });
        }

        function restartTest() {
            currentQuestion = 0;
            score = 0;
            userAnswers = [];
            showTest();
        }

        function backToSelection() {
            currentTest = null;
            currentQuestion = 0;
            score = 0;
            userAnswers = [];
            showTestSelection();
        }

        function logout() {
            currentUser = null;
            showLogin();
        }

        // Показать экран входа при загрузке
        showLogin();
    </script>
</body>
</html>
