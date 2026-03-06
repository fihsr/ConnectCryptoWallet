# ConnectCryptoWallet
Как подключать крипто-кошельки с помощью web3.js 

# Подключение крипто-кошельков (Web3-Onboarding)

В этом руководстве описан процесс интеграции криптовалейновых кошельков для сетей Ethereum (EVM) и Solana с использованием библиотеки Web3.js. Мы рассмотрим подключение MetaMask, Phantom и Solflare.

> **Важное замечание:** Библиотека `web3.js` (версии 1.x/4.x) предназначена для работы с **EVM-совместимыми** блокчейнами (Ethereum, Polygon, BSC и др.). Она **не работает** с блокчейном Solana напрямую. Для Solana мы будем использовать нативные SDK (Phantom/Solflare) или библиотеку `@solana/web3.js`. В примере ниже мы создадим универсальный интерфейс, который определит тип кошелька и подключит его правильным способом.

## Стек технологий

*   HTML, CSS, JavaScript (Ванильный JS для простоты примера)
*   [web3.js](https://web3js.readthedocs.io/) — для работы с MetaMask и другими EVM-кошельками
*   [@solana/web3.js](https://solana-labs.github.io/solana-web3.js/) — для работы с сетью Solana и Phantom/Solflare

## Содержание

1.  [Как это работает](#как-это-работает)
2.  [Установка зависимостей](#установка-зависимостей)
3.  [Детальный разбор кода](#детальный-разбор-кода)
    - [Подключение MetaMask (EVM)](#1-подключение-metamask-evm)
    - [Подключение Phantom (Solana)](#2-подключение-phantom-solana)
    - [Подключение Solflare (Solana)](#3-подключение-solflare-solana)
4.  [Полный пример (HTML + JS)](#полный-пример-html--js)
5.  [Обработка ошибок](#обработка-ошибок)

## Как это работает

1.  Пользователь выбирает кошелек (нажимает на кнопку "MetaMask", "Phantom" и т.д.).
2.  Скрипт проверяет, установлен ли соответствующий кошелек (`window.ethereum` для MetaMask, `window.solana` для Phantom и т.д.).
3.  Если кошелек найден, мы запрашиваем у пользователя разрешение на подключение (`eth_requestAccounts` или `connect()`).
4.  Получаем публичный адрес кошелька и, при необходимости, информацию о сети.
5.  Отображаем адрес пользователя на сайте (обрезанный для красоты) и кнопку отключения (дисконнекта).

## Установка зависимостей

Если вы используете сборщик (Webpack, Vite), установите библиотеки через npm:

```bash
npm install web3 @solana/web3.js
```

Для простого HTML-файла можно подключить библиотеки через CDN (как показано в полном примере ниже).

## Детальный разбор кода

### 1. Подключение MetaMask (EVM)

MetaMask внедряет объект `ethereum` в глобальный объект `window`. Мы используем Web3.js для взаимодействия с ним.

**Ключевые моменты:**
*   Проверка `typeof window.ethereum !== 'undefined'`.
*   Создание экземпляра `new Web3(window.ethereum)`.
*   Запрос аккаунтов через `window.ethereum.request({ method: 'eth_requestAccounts' })`.

```javascript
async function connectMetaMask() {
    if (typeof window.ethereum !== 'undefined') {
        try {
            const accounts = await window.ethereum.request({ method: 'eth_requestAccounts' });
            const web3 = new Web3(window.ethereum);
            
            const account = accounts[0];
            console.log('MetaMask подключен:', account);
            
            const balance = await web3.eth.getBalance(account);
            console.log('Баланс (Wei):', balance);
            
            return { address: account, provider: 'metamask', web3: web3 };
        } catch (error) {
            console.error('Пользователь отклонил подключение', error);
        }
    } else {
        alert('MetaMask не установлен! Скачайте его с https://metamask.io/');
    }
}
```

### 2. Подключение Phantom (Solana)

Phantom внедряет объект `solana` (если он является стандартным провайдером) или `phantom.solana`. Мы используем нативный метод `connect()`.

**Ключевые моменты:**
*   Проверка `window.solana && window.solana.isPhantom`.
*   Вызов `window.solana.connect()`.
*   Получение публичного ключа через `window.solana.publicKey.toString()`.
*   Для транзакций используется `@solana/web3.js`, но для простого подключения он не обязателен.

```javascript
async function connectPhantom() {
    if (window.solana && window.solana.isPhantom) {
        try {
            const response = await window.solana.connect();
            const publicKey = response.publicKey.toString();
            console.log('Phantom подключен:', publicKey);
            
            return { address: publicKey, provider: 'phantom' };
        } catch (error) {
            console.error('Ошибка подключения Phantom:', error);
        }
    } else {
        alert('Phantom не установлен! Скачайте его с https://phantom.app/');
    }
}
```

### 3. Подключение Solflare (Solana)

Solflare также внедряет объект, совместимый с Phantom API, но иногда использует пространство имен `solflare`. Однако, в большинстве случаев он также доступен через `window.solana` (если он выбран основным кошельком). Чтобы быть точнее, мы можем проверить `window.solflare` или использовать универсальный подход через `window.solana.isSolflare`.

```javascript
async function connectSolflare() {
    const provider = window.solflare || (window.solana?.isSolflare ? window.solana : null);
    
    if (provider) {
        try {
            if (window.solflare) {
                await window.solflare.connect();
                const publicKey = window.solflare.publicKey.toString();
                console.log('Solflare подключен:', publicKey);
                return { address: publicKey, provider: 'solflare' };
            } 
            else if (window.solana && window.solana.isSolflare) {
                const response = await window.solana.connect();
                const publicKey = response.publicKey.toString();
                console.log('Solflare (через solana) подключен:', publicKey);
                return { address: publicKey, provider: 'solflare' };
            }
        } catch (error) {
            console.error('Ошибка подключения Solflare:', error);
        }
    } else {
        alert('Solflare не установлен! Скачайте его с https://solflare.com/');
    }
}
```
