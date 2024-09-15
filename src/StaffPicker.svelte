<script>
  import { onMount } from "svelte";

  const SECRET =
    "n5v0k1DkLVuLzGR2uftfh9bR1woL8PL6RlHeJBtLEk0lsA6KsiGYwRL3PvKOb4yCXZVPtjEsDeoipRV9s3sJ4q21AEpWwbo5ncixVKqz6iW0kLIjq035M5etxu0xEZnd";

  $: tg = null;

  let username = "";
  let directories = {};

  let dateWork = "";
  let selectedDirector = 0;
  let selectedManager = 0;

  // Видимость кнопки сохранения
  $: tg && tg.MainButton
    ? (tg.MainButton.isVisible = /\d{4}-\d{2}-\d{2}/g.test(dateWork) && (selectedDirector > 0 || selectedManager > 0))
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

  async function sendServerData() {
    let data = {
      dt: `${dateWork}T00:00:00+03:00`,
      director_id: selectedDirector,
      manager_id: selectedManager,
    };

    fetch("/api/employees", {
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
    dateWork = formatDate(new Date());

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
      <h2 class="ml-3 text-3xl font-bold">Выбор сотрудников</h2>
    </div>

    <div class="m-1 text-2xl">
      <div class="mt-2 grid grid-cols-1 gap-1.5">
        <!-- Дата операции -->
        <label class="block">
          <span class="ml-1 text-gray-700">Дата</span>
          <input
            bind:value={dateWork}
            type="date"
            readonly
            class="mt-1 block h-10 w-full rounded border-gray-300 py-1 text-xl shadow-sm focus:border-gray-500 focus:ring focus:ring-gray-400 focus:ring-opacity-50"
          />
        </label>

        <!-- Директор -->
        <label class="block">
          <span class="ml-1 text-gray-700">Директор</span>
          <select
            bind:value={selectedDirector}
            class="mt-1 block w-full rounded border-gray-300 py-1 text-xl shadow-sm focus:border-gray-500 focus:ring focus:ring-gray-400 focus:ring-opacity-50"
          >
            <option value="" disabled selected>Выберите директора...</option>
            {#if directories && directories.users}
              {#each directories.users.filter((u) => u.is_director) as employee}
                <option value={employee.id}>{employee.name}</option>
              {/each}
            {/if}
          </select>
        </label>

        <!-- Менеджер -->
        <label class="block">
          <span class="ml-1 text-gray-700">Менеджер</span>
          <select
            bind:value={selectedManager}
            class="mt-1 block w-full rounded border-gray-300 py-1 text-xl shadow-sm focus:border-gray-500 focus:ring focus:ring-gray-400 focus:ring-opacity-50"
          >
            <option value="" disabled selected>Выберите менеджера...</option>
            {#if directories && directories.users}
              {#each directories.users.filter((u) => u.is_manager) as employee}
                <option value={employee.id}>{employee.name}</option>
              {/each}
            {/if}
          </select>
        </label>
      </div>
    </div>
  </div>
</main>
