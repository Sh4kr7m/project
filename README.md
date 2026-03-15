Проект доступен по ссылке https://jayflow.ai/api/webapp/9a8375f4-8772-4296-9bcb-521538349400.html


Описание принципа работы гонки

  ПРОВЕРКА ГОНКИ (Race Condition Check):
  Чтобы проверить корректность работы при параллельных запросах:
  1. Откройте приложение в двух разных вкладках браузера.
  2. В обеих вкладках выберите роль "Диспетчер" и убедитесь, что есть заявка в статусе "Новая" (или создайте её).
  3. В ПЕРВОЙ вкладке назначьте мастера (например, Иванов).
  4. Перейдите во ВТОРУЮ вкладку, выберите мастера Иванова. Вы увидите ту же заявку.
  5. Во второй вкладке вызовите консоль разработчика (F12) и найдите кнопку "Взять в работу". 
  6. Чтобы имитировать одновременный запрос вручную: 
     Кликните кнопку "Взять в работу" одновременно в двух вкладках (либо во второй вкладке,
     если в первой уже нажали, но "сервер" еще не ответил).
  7. В коде установлена задержка MockServer.transitionToWork в 800мс.
  8. Первая вкладка получит успех (200), а вторая (даже если кнопка была доступна на экране) 
     получит статус 409 Conflict, так как состояние в хранилище уже изменилось.
ВТорой вариант 
Как работает защита от гонки:
Откройте две вкладки и в обеих выберите мастера Иванова
В первой вкладке нажмите "Взять в работу"
Быстро переключитесь во вторую вкладку и тоже нажмите "Взять в работу" (пока первая еще обрабатывается)
Первая вкладка получит успех (200)
Вторая вкладка получит 409 Conflict с сообщением о том, что заявка уже занята
Благодаря storage event, вторая вкладка автоматически обновится и покажет заявку в статусе "В работе"





Тест 1: Проверка защиты от Race Condition
javascript
// test-race-condition.js
// Запускать в браузерной консоли при открытых двух вкладках с ролью мастера

async function testRaceCondition() {
    console.log('🧪 Запуск теста защиты от Race Condition...');
    
    // Получаем ID первой заявки в статусе 'assigned'
    const requests = JSON.parse(localStorage.getItem('srv_requests_v2'));
    const targetRequest = requests.find(r => r.status === 'assigned');
    
    if (!targetRequest) {
        console.error('❌ Нет заявки в статусе "assigned". Создайте тестовые данные.');
        return;
    }
    
    console.log(`🎯 Целевая заявка ID: ${targetRequest.id}, текущий статус: ${targetRequest.status}`);
    
    // Имитируем два параллельных запроса на взятие в работу
    const mockServer1 = new MockServer(); // В реальном коде используем существующий
    const mockServer2 = new MockServer();
    
    // Запускаем параллельно
    const [result1, result2] = await Promise.all([
        mockServer1.transitionToWork(targetRequest.id),
        mockServer2.transitionToWork(targetRequest.id)
    ]);
    
    console.log('📊 Результат первого запроса:', result1);
    console.log('📊 Результат второго запроса:', result2);
    
    // Проверяем, что один успешен, второй с конфликтом
    if (result1.status === 200 && result2.status === 409) {
        console.log('✅ ТЕСТ ПРОЙДЕН: Race condition предотвращен, второй запрос получил 409');
    } else if (result2.status === 200 && result1.status === 409) {
        console.log('✅ ТЕСТ ПРОЙДЕН: Race condition предотвращен, первый запрос получил 409');
    } else {
        console.error('❌ ТЕСТ НЕ ПРОЙДЕН: Ожидался один 200 и один 409, получены:', result1.status, result2.status);
    }
    
    // Проверяем финальный статус в хранилище
    const finalRequests = JSON.parse(localStorage.getItem('srv_requests_v2'));
    const finalRequest = finalRequests.find(r => r.id === targetRequest.id);
    console.log(`📌 Финальный статус заявки: ${finalRequest.status} (должен быть "in_progress")`);
    
    if (finalRequest.status === 'in_progress') {
        console.log('✅ Статус корректно обновлен на "in_progress"');
    } else {
        console.error('❌ Ошибка: статус не обновился');
    }
}

// Для запуска вручную:
// testRaceCondition();


Тест 2: Проверка синхронизации между вкладками
javascript
// test-cross-tab-sync.js
// Откройте две вкладки и выполните этот код в консоли ПЕРВОЙ вкладки

async function testCrossTabSync() {
    console.log('🧪 Запуск теста синхронизации между вкладками...');
    console.log('📌 Инструкция:');
    console.log('1. Откройте вторую вкладку с этим приложением');
    console.log('2. Во второй вкладке выберите роль "Диспетчер"');
    console.log('3. Нажмите Enter в этой консоли для продолжения');
    
    await new Promise(resolve => {
        document.addEventListener('keydown', function handler(e) {
            if (e.key === 'Enter') {
                document.removeEventListener('keydown', handler);
                resolve();
            }
        });
        console.log('⏎ Нажмите Enter...');
    });
    
    // Создаем новую заявку в первой вкладке
    const newRequest = {
        id: Date.now(),
        clientName: "Тестовый Клиент",
        phone: "+7 (999) 123-45-67",
        address: "ул. Тестовая, д. 1",
        problemText: "Автоматический тест синхронизации",
        status: "new",
        assignedTo: "",
        createdAt: new Date().toISOString(),
        updatedAt: new Date().toISOString()
    };
    
    const requests = JSON.parse(localStorage.getItem('srv_requests_v2')) || [];
    requests.push(newRequest);
    localStorage.setItem('srv_requests_v2', JSON.stringify(requests));
    
    console.log('✅ Заявка создана в ПЕРВОЙ вкладке, ID:', newRequest.id);
    console.log('📢 Проверьте ВТОРУЮ вкладку - там должно появиться уведомление о синхронизации');
    console.log('⏳ Ожидание 3 секунды...');
    
    await new Promise(r => setTimeout(r, 3000));
    
    // Проверяем, обновилась ли вторая вкладка (через storage event)
    console.log('🔍 Проверка... Если вы видели уведомление во второй вкладке - синхронизация работает');
    
    // Модифицируем заявку из первой вкладки
    const updatedRequests = JSON.parse(localStorage.getItem('srv_requests_v2'));
    const targetReq = updatedRequests.find(r => r.id === newRequest.id);
    targetReq.status = 'assigned';
    targetReq.assignedTo = 'Иванов';
    localStorage.setItem('srv_requests_v2', JSON.stringify(updatedRequests));
    
    console.log('✅ Статус заявки изменен на "assigned" в ПЕРВОЙ вкладке');
    console.log('📢 Во ВТОРОЙ вкладке должно появиться второе уведомление');
    
    // Финальная проверка
    setTimeout(() => {
        console.log('🎉 Тест завершен. Если во второй вкладке были уведомления - синхронизация работает корректно.');
    }, 2000);
}

// Для запуска:
// testCrossTabSync();
Инструкция по запуску тестов:
Откройте приложение в браузере

Откройте инструменты разработчика (F12)

Перейдите на вкладку Console

Скопируйте код нужного теста

Вставьте в консоль и нажмите Enter

Следуйте инструкциям в консоли

Ожидаемые результаты:

Тест 1 должен показать один успешный ответ (200) и один конфликт (409)

Тест 2 должен продемонстрировать автоматическое обновление данных во второй вкладке

