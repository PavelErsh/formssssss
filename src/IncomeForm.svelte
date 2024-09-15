<script>
  import { onMount, beforeUpdate } from "svelte";

  let visiblePopup = false;

  const formatter = new Intl.NumberFormat("ru");
  const SECRET =
    "n5v0k1DkLVuLzGR2uftfh9bR1woL8PL6RlHeJBtLEk0lsA6KsiGYwRL3PvKOb4yCXZVPtjEsDeoipRV9s3sJ4q21AEpWwbo5ncixVKqz6iW0kLIjq035M5etxu0xEZnd";

  $: tg = null;

  let username = "";
  $: minDate = "";
  let maxDate = formatDate(new Date());
  let directories = {};

  let dateIncome = "";
  $: selectedFlightNumber = "1";
  let selectedInstructor = 0;
  let selectedRoute = 0;

  let techniques = [
    {
      typeTechnique: 0,
      discount: null,
      prepayed: false,
      typePayment: 0,
      clientSource: 0,
      hotel: 0,
      transfer: null,
      description: "",
      amount: 0,
    },
  ];

  $: {
    if (dateIncome) {
      getMinDate();
      if (dateIncome < minDate) dateIncome = minDate;
      else if (dateIncome > maxDate) dateIncome = maxDate;
    }
  }

  $: dateIncome ? getFlightNumber(dateIncome) : null;

  // Видимость кнопки сохранения
  $: tg && tg.MainButton
    ? (tg.MainButton.isVisible =
        /\d{4}-\d{2}-\d{2}/g.test(dateIncome) &&
        selectedInstructor > 0 &&
        selectedRoute > 0 &&
        techniques.every((t) => t.typeTechnique > 0 && t.typePayment > 0 && t.clientSource > 0))
    : null;

  function addTechique() {
    techniques.push({
      typeTechnique: 0,
      discount: null,
      prepayed: false,
      typePayment: 0,
      clientSource: 0,
      hotel: 0,
      transfer: null,
      description: "",
      amount: 0,
    });

    techniques = techniques;
  }

  function addDoubleTechique() {
    const lastTech = techniques[techniques.length - 1];

    techniques.push({
      typeTechnique: lastTech.typeTechnique,
      discount: lastTech.discount,
      prepayed: lastTech.prepayed,
      typePayment: lastTech.typePayment,
      clientSource: lastTech.clientSource,
      hotel: lastTech.hotel,
      transfer: null,
      description: "",
      amount: lastTech.amount,
    });

    techniques = techniques;
  }

  function removeTechique(index) {
    // Первый не удаляем
    if (index == 0) return;
    if (index == techniques.length - 1) {
      techniques = techniques.slice(0, techniques.length - 1);
    } else {
      techniques = techniques.slice(0, index).concat(techniques.slice(index + 1));
    }
  }

  function formatDate(date) {
    let d = new Date(date),
      month = "" + (d.getMonth() + 1),
      day = "" + d.getDate(),
      year = d.getFullYear();

    if (month.length < 2) month = "0" + month;
    if (day.length < 2) day = "0" + day;

    return [year, month, day].join("-");
  }

  async function getServerDirectories() {
    fetch("/api/directories", {
      method: "POST",
      headers: { "Content-Type": "application/json;charset=utf-8" },
      body: JSON.stringify({
        key: SECRET,
      }),
    })
      .then((response) => {
        return response.json();
      })
      .then((data) => {
        directories = data;
      })
      .catch((err) => {
        console.error(err);
      });
  }

  async function getMinDate() {
    fetch("/api/flight_min_date", {
      method: "POST",
      headers: { "Content-Type": "application/json;charset=utf-8" },
      body: JSON.stringify({
        key: SECRET,
      }),
    })
      .then((response) => {
        return response.json();
      })
      .then((data) => {
        minDate = data.date;
      })
      .catch((err) => {
        console.error(err);
      });
  }

  async function getFlightNumber(dt) {
    if (username) {
      fetch("/api/flight_number", {
        method: "POST",
        headers: { "Content-Type": "application/json;charset=utf-8" },
        body: JSON.stringify({
          key: SECRET,
          date: dt,
          username: username,
        }),
      })
        .then((response) => {
          return response.json();
        })
        .then((data) => {
          selectedFlightNumber = data.flight_number;
        })
        .catch((err) => {
          console.error(err);
        });
    }
  }

  async function sendServerData() {
    let data = {
      dateIncome: dateIncome,
      typeRoute: selectedRoute,
      flightNumber: Number(selectedFlightNumber),
      instructor: selectedInstructor,
      techniques: [],
    };

    techniques.forEach((t) => {
      data.techniques.push({
        typeTechnique: t.typeTechnique,
        discount: t.discount ? Math.abs(t.discount) : 0,
        prepayment: t.prepayed,
        price: t.amount,
        typePayment: t.typePayment,
        clientSource: t.clientSource,
        hotel: t.hotel,
        transfer: t.transfer ? Math.abs(t.transfer) : 0,
        description: t.description,
      });
    });

    fetch("/api/deposit_income", {
      method: "POST",
      headers: { "Content-Type": "application/json;charset=utf-8" },
      body: JSON.stringify({
        key: SECRET,
        value: tg.initData,
        data: data,
      }),
    })
      .catch((err) => {
        console.error(err);
        visiblePopup = true;
      })
      .finally(() => tg.close());
  }

  // Загружаем данные при создании формы
  onMount(() => {
    getServerDirectories();
    getMinDate();
    dateIncome = formatDate(new Date());

    tg = window.Telegram.WebApp;

    if (tg && tg.initDataUnsafe && tg.initDataUnsafe.user) {
      username = tg.initDataUnsafe.user.username;
    }

    if (tg && tg.MainButton) {
      tg.MainButton.setText("Сохранить");
      tg.MainButton.color = "#a3e635";
      tg.backgroundColor = "#e5e7eb";
      tg.headerColor = "#e5e7eb";
    }

    tg.onEvent("mainButtonClicked", () => sendServerData());
    tg.enableClosingConfirmation();
  });

  beforeUpdate(() => {
    if (selectedRoute) {
      techniques.forEach((t) => {
        // console.log(t);
        if (t.typeTechnique) {
          let findTech = directories.techniques.find((tech) => tech.id == t.typeTechnique);
          // console.log(findTech);

          if (findTech) {
            let findRoute = directories.prices.find(
              (route) => route.type_id == findTech.type_id && route.route_id == selectedRoute
            );
            // console.log(findRoute);

            if (findRoute) {
              t.amount =
                findRoute.cost -
                (t.discount ? (isNaN(Number(t.discount)) ? 0 : Math.abs(Number(t.discount))) : 0) -
                (t.transfer ? (isNaN(Number(t.transfer)) ? 0 : Math.abs(Number(t.transfer))) : 0) -
                (t.prepayed ? 1000 : 0);
            }
          }
        }
      });
    }
  });
</script>

<main class="max-w-lg bg-gray-200">
  <div class="pb-64 w-full px-3">
    <div class="m-1">
      <h2 class="ml-3 text-3xl font-bold">Доход</h2>
    </div>

    <div class="m-1 text-2xl">
      <div class="mt-2 grid grid-cols-1 gap-1.5">
        <div class="grid grid-cols-2 gap-x-1">
          <!-- Дата дохода -->
          <label class="block">
            <span class="ml-1 text-gray-700">Дата</span>
            <input
              bind:value={dateIncome}
              type="date"
              class="mt-1 block h-10 w-full rounded border-gray-300 py-1 text-xl shadow-sm focus:border-gray-500 focus:ring focus:ring-gray-400 focus:ring-opacity-50"
            />
          </label>

          <!-- Номер рейса -->
          <label class="block">
            <span class="ml-1 text-gray-700">Номер рейса</span>
            <select
              bind:value={selectedFlightNumber}
              class="mt-1 block h-10 w-full rounded border-gray-300 py-1 text-xl shadow-sm focus:border-gray-500 focus:ring focus:ring-gray-400 focus:ring-opacity-50"
            >
              {#each ["1", "2", "3", "4", "5", "6", "7", "8", "9", "10", "11", "12", "13", "14", "15", "16", "17", "18", "19", "20"] as r}
                <option value={r}>{r}</option>
              {/each}
            </select>
          </label>
        </div>

        <!-- Инструктор -->
        <label class="block">
          <span class="ml-1 text-gray-700">Инструктор</span>
          <select
            bind:value={selectedInstructor}
            class="mt-1 block w-full rounded border-gray-300 py-1 text-xl shadow-sm focus:border-gray-500 focus:ring focus:ring-gray-400 focus:ring-opacity-50"
          >
            <option value="" disabled selected></option>
            {#if directories && directories.users}
              {#each directories.users as manager}
                {#if manager.is_instructor}
                  <option value={manager.id}>{manager.name}</option>
                {/if}
              {/each}
            {/if}
          </select>
        </label>

        <!-- Маршрут -->
        <label class="block">
          <span class="ml-1 text-gray-700">Вид маршрута</span>
          <select
            bind:value={selectedRoute}
            class="mt-1 block w-full rounded border-gray-300 py-1 text-xl shadow-sm focus:border-gray-500 focus:ring focus:ring-gray-400 focus:ring-opacity-50"
          >
            <option value="" disabled selected></option>
            {#if directories && directories.routes}
              {#each directories.routes as route}
                <option value={route.id}>{route.name}</option>
              {/each}
            {/if}
          </select>
        </label>
      </div>

      <h3 class="ml-3 mt-2 text-2xl font-bold">Техника:</h3>
      {#if techniques}
        {#each techniques as t, i}
          <span class="mt-1 flex items-center">
            <span class="h-1 flex-1 bg-gray-400"></span>
            <span class="shrink-0 px-6 text-gray-400">Техника №{i + 1} ({formatter.format(t.amount)} ₽)</span>
            <span class="h-1 flex-1 bg-gray-400"></span>
            {#if i > 0}
              <button
                on:click={() => removeTechique(i)}
                class="ml-1 rounded-xl text-white bg-red-400 border-red-700 border-2 hover:bg-red-500"
              >
                <svg viewBox="0 0 24 24" class="h-8 w-8"
                  ><path
                    fill="currentColor"
                    d="m8.382 17.025l-1.407-1.4L10.593 12L6.975 8.4L8.382 7L12 10.615L15.593 7L17 8.4L13.382 12L17 15.625l-1.407 1.4L12 13.41l-3.618 3.615Z"
                  /></svg
                >
              </button>
            {/if}
          </span>

          <div class="mt-2 grid grid-cols-1 gap-1.5">
            <!-- Вид техники -->
            <label class="block">
              <span class="ml-1 text-gray-700">Вид техники</span>
              <select
                bind:value={t.typeTechnique}
                class="mt-1 block w-full rounded border-gray-300 py-1 text-xl shadow-sm focus:border-gray-500 focus:ring focus:ring-gray-400 focus:ring-opacity-50"
              >
                <option value="" disabled selected></option>
                {#if directories && directories.techniques}
                  {#each directories.type_techniques as typeTech, index}
                    <optgroup label={typeTech.name} class={index == 1 ? "bg-orange-100" : index == 2 ? "bg-violet-100" : ""}>
                      {#each directories.techniques.filter((t) => t.type_id == typeTech.id) as tech}
                        <option value={tech.id}>{tech.name}</option>
                      {/each}
                    </optgroup>
                  {/each}
                {/if}
              </select>
            </label>

            <!-- Размер скидки -->
            <label class="block">
              <span class="ml-1 text-gray-700">Размер скидки</span>
              <input
                bind:value={t.discount}
                type="number"
                class="mt-1 block w-full rounded border-gray-300 py-1 text-xl shadow-sm focus:border-gray-500 focus:ring focus:ring-gray-400 focus:ring-opacity-50"
              />
            </label>

            <!-- Предоплата -->
            <div class="mt-2">
              <label class="inline-flex items-center">
                <input
                  bind:checked={t.prepayed}
                  type="checkbox"
                  class="h-9 w-9 rounded border-gray-300 pl-2 text-2xl text-gray-700 shadow-sm focus:border-gray-500 focus:ring focus:ring-gray-400 focus:ring-opacity-50 focus:ring-offset-0"
                />
                <span class="ml-2 text-2xl">Предоплата</span>
              </label>
            </div>

            <!-- Вид оплаты -->
            <label class="block">
              <span class="ml-1 text-gray-700">Вид оплаты</span>
              <select
                bind:value={t.typePayment}
                class="mt-1 block w-full rounded border-gray-300 py-1 text-xl shadow-sm focus:border-gray-500 focus:ring focus:ring-gray-400 focus:ring-opacity-50"
              >
                <option value="" disabled selected></option>
                {#if directories && directories.pays}
                  {#each directories.pays as pay}
                    <option value={pay.id}>{pay.name}</option>
                  {/each}
                {/if}
              </select>
            </label>

            <!-- Источник клиента -->
            <label class="block">
              <span class="ml-1 text-gray-700">Источник клиента</span>
              <select
                bind:value={t.clientSource}
                class="mt-1 block w-full rounded border-gray-300 py-1 text-xl shadow-sm focus:border-gray-500 focus:ring focus:ring-gray-400 focus:ring-opacity-50"
              >
                <option value="" disabled selected></option>
                {#if directories && directories.sources}
                  {#each directories.sources as source}
                    <option value={source.id}>{source.name}</option>
                  {/each}
                {/if}
              </select>
            </label>

            <!-- Выбор отеля -->
            {#if t.clientSource == 3}
              <label class="block">
                <span class="ml-1 text-gray-700">Отель</span>
                <select
                  bind:value={t.hotel}
                  class="mt-1 block w-full rounded border-gray-300 py-1 text-xl shadow-sm focus:border-gray-500 focus:ring focus:ring-gray-400 focus:ring-opacity-50"
                >
                  <option value="" disabled selected></option>
                  {#if directories && directories.hotels}
                    {#each directories.hotels as hotel}
                      <option value={hotel.id}>{hotel.name}</option>
                    {/each}
                  {/if}
                </select>
              </label>
            {/if}

            <!-- Трансфер -->
            <label class="block">
              <span class="ml-1 text-gray-700">Трансфер</span>
              <input
                bind:value={t.transfer}
                type="number"
                class="mt-1 block w-full rounded border-gray-300 py-1 text-xl shadow-sm focus:border-gray-500 focus:ring focus:ring-gray-400 focus:ring-opacity-50"
              />
            </label>

            <!-- Примечание -->
            <label class="block">
              <span class="ml-1 text-gray-700">Комментарий</span>
              <textarea
                bind:value={t.description}
                class="mt-1 block w-full rounded border-gray-300 py-1 text-xl shadow-sm focus:border-gray-500 focus:ring focus:ring-gray-400 focus:ring-opacity-50"
                rows="3"
              />
            </label>
          </div>
        {/each}
      {/if}

      <!-- Кнопки добавления -->
      <div class="mt-3 grid grid-cols-2 gap-x-1">
        <button
          on:click={() => addDoubleTechique()}
          class="rounded-xl border border-indigo-500 bg-indigo-300 pb-2 pt-1 hover:border-indigo-600 hover:bg-indigo-400 disabled:border-gray-500 disabled:bg-gray-300 disabled:hover:border-gray-500 disabled:hover:bg-gray-300"
        >
          <div class="flex flex-row px-2 items-center justify-between">
            <svg viewBox="0 0 14 14" class="h-8 w-8"
              ><g fill="none" stroke="currentColor" stroke-linecap="round" stroke-linejoin="round"
                ><path
                  d="M5.513 4.963H4.77a1 1 0 0 0-1 .68L3 7.963H1.5a1 1 0 0 0-1 1v2a1 1 0 0 0 1 1H2m10-4h.5a1 1 0 0 1 1 1v2a1 1 0 0 1-1 1H12"
                /><path
                  d="M10.499 13.463a1.501 1.501 0 1 1 0-3.002a1.501 1.501 0 0 1 0 3.002Zm-7 0a1.501 1.501 0 1 1 0-3.002a1.501 1.501 0 0 1 0 3.002Zm5.499-1.5H5M7.416 3.97c-.351-.06-.351-.564 0-.625A3.176 3.176 0 0 0 9.974.895l.022-.097c.075-.347.57-.349.648-.003l.026.113a3.193 3.193 0 0 0 2.565 2.435c.353.062.353.568 0 .63a3.193 3.193 0 0 0-2.565 2.435l-.026.112c-.079.347-.572.344-.648-.002l-.022-.097a3.176 3.176 0 0 0-2.558-2.45Z"
                /></g
              ></svg
            >
            <span class="text-xl font-bold">Дублировать</span>
          </div>
        </button>

        <button
          on:click={() => addTechique()}
          class="rounded-xl border border-blue-500 bg-blue-300 pb-2 pt-1 hover:border-blue-600 hover:bg-blue-400 disabled:border-gray-500 disabled:bg-gray-300 disabled:hover:border-gray-500 disabled:hover:bg-gray-300"
        >
          <div class="flex flex-row px-2 items-center justify-between">
            <svg viewBox="0 0 14 14" class="h-8 w-8"
              ><g fill="none" stroke="currentColor" stroke-linecap="round" stroke-linejoin="round"
                ><path
                  d="M5.513 4.963H4.77a1 1 0 0 0-1 .68L3 7.963H1.5a1 1 0 0 0-1 1v2a1 1 0 0 0 1 1H2m10-4h.5a1 1 0 0 1 1 1v2a1 1 0 0 1-1 1H12"
                /><path
                  d="M10.499 13.463a1.501 1.501 0 1 1 0-3.002a1.501 1.501 0 0 1 0 3.002Zm-7 0a1.501 1.501 0 1 1 0-3.002a1.501 1.501 0 0 1 0 3.002Zm5.499-1.5H5M7.416 3.97c-.351-.06-.351-.564 0-.625A3.176 3.176 0 0 0 9.974.895l.022-.097c.075-.347.57-.349.648-.003l.026.113a3.193 3.193 0 0 0 2.565 2.435c.353.062.353.568 0 .63a3.193 3.193 0 0 0-2.565 2.435l-.026.112c-.079.347-.572.344-.648-.002l-.022-.097a3.176 3.176 0 0 0-2.558-2.45Z"
                /></g
              ></svg
            >
            <span class="text-xl font-bold">Добавить</span>
          </div>
        </button>
      </div>
    </div>
  </div>
</main>

{#if visiblePopup}
  <div role="alert" class="absolute top-5 left-3 right-3 rounded-xl border border-red-300 bg-red-100 shadow-lg p-4">
    <div class="flex items-start gap-4">
      <span class="text-red-600">
        <svg
          xmlns="http://www.w3.org/2000/svg"
          fill="none"
          viewBox="0 0 24 24"
          stroke-width="1.5"
          stroke="currentColor"
          class="h-12 w-12"
        >
          <path stroke-linecap="round" stroke-linejoin="round" d="M9.401 3.003c1.155-2 4.043-2 5.197 0l7.355 12.748c1.154 2-.29 4.5-2.599 4.5H4.645c-2.309 0-3.752-2.5-2.598-4.5L9.4 3.003zM12 8.25a.75.75 0 01.75.75v3.75a.75.75 0 01-1.5 0V9a.75.75 0 01.75-.75zm0 8.25a.75.75 0 100-1.5.75.75 0 000 1.5z" />
        </svg>
      </span>

      <div class="flex-1">
        <strong class="block text-3xl font-medium text-gray-900">Ошибка сохранения данных!</strong>

        <p class="mt-1 text-xl text-gray-700">Внесение данных после 18:00 не доступно!</p>
      </div>

      <button on:click={() => (visiblePopup = false)} class="text-gray-500 transition hover:text-gray-600">
        <span class="sr-only">Закрыть сообщение</span>

        <svg
          xmlns="http://www.w3.org/2000/svg"
          fill="none"
          viewBox="0 0 24 24"
          stroke-width="1.5"
          stroke="currentColor"
          class="h-12 w-12"
        >
          <path stroke-linecap="round" stroke-linejoin="round" d="M6 18L18 6M6 6l12 12" />
        </svg>
      </button>
    </div>
  </div>
{/if}
