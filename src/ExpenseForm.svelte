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

  let dateExpense = "";
  let selectedExpense = 0;
  let selectedInvestor = 0;
  let selectedEmployee = 0;
  let amount = null;
  let description = "";

  $: visibleInvestor =
    directories && directories.type_expenses && directories.type_expenses.find((r) => r.id == selectedExpense)
      ? directories.type_expenses.find((r) => r.id == selectedExpense).visible_investor
      : false;
  $: visibleEmployee =
    directories && directories.type_expenses && directories.type_expenses.find((r) => r.id == selectedExpense)
      ? directories.type_expenses.find((r) => r.id == selectedExpense).visible_employee
      : false;

  $: {
    if (dateExpense) {
      getMinDate();
      if (dateExpense < minDate) dateExpense = minDate;
      else if (dateExpense > maxDate) dateExpense = maxDate;
    }
  }

  // Видимость кнопки сохранения
  $: tg && tg.MainButton
    ? (tg.MainButton.isVisible =
        /\d{4}-\d{2}-\d{2}/g.test(dateExpense) &&
        selectedExpense &&
        amount && amount > 0 &&
        ((!visibleInvestor || (visibleInvestor && selectedInvestor > 0)) && (!visibleEmployee || (visibleEmployee && selectedEmployee > 0))))
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
      dt: `${dateExpense}T00:00:00+03:00`,
      type_expense_id: selectedExpense,
      type_tech_id: selectedInvestor,
      employee_id: selectedEmployee,
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
    dateExpense = formatDate(new Date());

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
      <h2 class="ml-3 text-3xl font-bold">Расход</h2>
    </div>

    <div class="m-1 text-2xl">
      <div class="mt-2 grid grid-cols-1 gap-1.5">
        <!-- Дата операции -->
        <label class="block">
          <span class="ml-1 text-gray-700">Дата</span>
          <input
            bind:value={dateExpense}
            type="date"
            class="mt-1 block h-10 w-full rounded border-gray-300 py-1 text-xl shadow-sm focus:border-gray-500 focus:ring focus:ring-gray-400 focus:ring-opacity-50"
          />
        </label>

        <!-- Вид операции -->
        <label class="block">
          <span class="ml-1 text-gray-700">Вид расхода</span>
          <select
            bind:value={selectedExpense}
            class="mt-1 block w-full rounded border-gray-300 py-1 text-xl shadow-sm focus:border-gray-500 focus:ring focus:ring-gray-400 focus:ring-opacity-50"
          >
            <option value="" disabled selected></option>
            {#if directories && directories.cat_expenses}
              {#each directories.cat_expenses as cat_exp}
              <optgroup label={cat_exp.cat}>
                {#if directories && directories.type_expenses}
                  {#each directories.type_expenses.filter(te => te.cat_id == cat_exp.id) as type_exp}
                    <option value={type_exp.id}>{type_exp.name}</option>
                  {/each}
                {/if}
              </optgroup>
              {/each}
            {/if}
          </select>
        </label>

        <!-- Инвестор -->
        {#if visibleInvestor}
          <label class="block">
            <span class="ml-1 text-gray-700">Инвестор</span>
            <select
              bind:value={selectedInvestor}
              class="mt-1 block w-full rounded border-gray-300 py-1 text-xl shadow-sm focus:border-gray-500 focus:ring focus:ring-gray-400 focus:ring-opacity-50"
            >
              <option value="" disabled selected></option>
              {#if directories && directories.investors}
                {#each directories.investors as investor}
                  <option value={investor.id}>{investor.name}</option>
                {/each}
              {/if}
            </select>
          </label>
        {/if}

        <!-- Сотрудник -->
        {#if visibleEmployee}
          <label class="block">
            <span class="ml-1 text-gray-700">Сотрудник</span>
            <select
              bind:value={selectedEmployee}
              class="mt-1 block w-full rounded border-gray-300 py-1 text-xl shadow-sm focus:border-gray-500 focus:ring focus:ring-gray-400 focus:ring-opacity-50"
            >
              <option value="" disabled selected></option>
              {#if directories && directories.users}
                {#each directories.users.filter((u) => u.is_director || u.is_manager || u.is_instructor || u.is_assistant) as employee}
                  <option value={employee.id}>{employee.name}</option>
                {/each}
              {/if}
            </select>
          </label>
        {/if}

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
