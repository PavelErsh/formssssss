<script>
  import { onMount } from "svelte";

  // const formatter = new Intl.NumberFormat("ru");
  const SECRET =
    "n5v0k1DkLVuLzGR2uftfh9bR1woL8PL6RlHeJBtLEk0lsA6KsiGYwRL3PvKOb4yCXZVPtjEsDeoipRV9s3sJ4q21AEpWwbo5ncixVKqz6iW0kLIjq035M5etxu0xEZnd";

  $: tg = null;

  let username = "";
  $: minDate = "";
  let maxDate = formatDate(new Date());
  let directories = {};

  let dateComission = "";
  let selectedComission = 0;
  let amount = null;
  let description = "";

  $: {
    if (dateComission) {
      getMinDate();
      if (dateComission < minDate) dateComission = minDate;
      else if (dateComission > maxDate) dateComission = maxDate;
    }
  }

  // Видимость кнопки сохранения
  $: tg && tg.MainButton
    ? (tg.MainButton.isVisible = /\d{4}-\d{2}-\d{2}/g.test(dateComission) && selectedComission && amount && amount > 0)
    : null;

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
    fetch("/api/exp_min_date", {
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

  async function sendServerData() {
    let data = {
      dt: `${dateComission}T00:00:00+03:00`,
      type_expense_id: selectedComission,
      type_tech_id: 0,
      employee_id: 0,
      amount: amount ? amount : 0,
      note: description,
      source_id: 1,
    };

    fetch("/api/expense", {
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
      })
      .finally(() => tg.close());
  }

  // Загружаем данные при создании формы
  onMount(() => {
    getServerDirectories();
    getMinDate();
    dateComission = formatDate(new Date());

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
</script>

<main class="max-w-lg bg-gray-200">
  <div class="pb-64 w-full px-3">
    <div class="m-1">
      <h2 class="ml-3 text-3xl font-bold">Комиссия</h2>
    </div>

    <div class="m-1 text-2xl">
      <div class="mt-2 grid grid-cols-1 gap-1.5">
        <!-- Дата операции -->
        <label class="block">
          <span class="ml-1 text-gray-700">Дата</span>
          <input
            bind:value={dateComission}
            type="date"
            class="mt-1 block h-10 w-full rounded border-gray-300 py-1 text-xl shadow-sm focus:border-gray-500 focus:ring focus:ring-gray-400 focus:ring-opacity-50"
          />
        </label>

        <!-- Вид операции -->
        <label class="block">
          <span class="ml-1 text-gray-700">Вид комисии</span>
          <select
            bind:value={selectedComission}
            class="mt-1 block w-full rounded border-gray-300 py-1 text-xl shadow-sm focus:border-gray-500 focus:ring focus:ring-gray-400 focus:ring-opacity-50"
          >
            <option value="" disabled selected></option>
            {#if directories && directories.cat_expenses}
              {#each directories.cat_expenses.filter((ce) => ce.cat == "Выплаты") as cat_exp}
                {#if directories && directories.type_expenses}
                  {#each directories.type_expenses.filter((te) => te.cat_id == cat_exp.id && (te.name == "Проценты агентам" || te.name == "Кешбек за отзывы")) as type_exp}
                    <option value={type_exp.id}>{type_exp.name}</option>
                  {/each}
                {/if}
              {/each}
            {/if}
          </select>
        </label>

        <!-- Сумма -->
        <label class="block">
          <span class="ml-1 text-gray-700">Сумма</span>
          <input
            bind:value={amount}
            type="number"
            class="mt-1 block w-full rounded border-gray-300 py-1 text-xl shadow-sm focus:border-gray-500 focus:ring focus:ring-gray-400 focus:ring-opacity-50"
          />
        </label>

        <!-- Примечание -->
        <label class="block">
          <span class="ml-1 text-gray-700">Комментарий</span>
          <textarea
            bind:value={description}
            class="mt-1 block w-full rounded border-gray-300 py-1 text-xl shadow-sm focus:border-gray-500 focus:ring focus:ring-gray-400 focus:ring-opacity-50"
            rows="3"
          />
        </label>
      </div>
    </div>
  </div>
</main>
