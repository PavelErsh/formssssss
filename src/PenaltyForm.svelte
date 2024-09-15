<script>
  import { onMount } from "svelte";

  const SECRET =
    "n5v0k1DkLVuLzGR2uftfh9bR1woL8PL6RlHeJBtLEk0lsA6KsiGYwRL3PvKOb4yCXZVPtjEsDeoipRV9s3sJ4q21AEpWwbo5ncixVKqz6iW0kLIjq035M5etxu0xEZnd";

  $: tg = null;

  let maxDate = formatDate(new Date());
  let directories = {};

  let dateOperation = "";
  let amount = null;
  let description = "";

  // Видимость выпадающего списка с артикулами
  $: visibleDropdownList = false;

  $: {
    if (dateOperation) {
      if (dateOperation > maxDate) dateOperation = maxDate;
    }
  }

  // Видимость кнопки сохранения
  $: tg && tg.MainButton
    ? (tg.MainButton.isVisible =
        /\d{4}-\d{2}-\d{2}/g.test(dateOperation) && amount && amount > 0 && directories.users.some((u) => u.checked))
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
        directories.users = [];

        data.users.forEach((d, i) => {
          directories.users.push({
            id: d.id,
            name: d.name,
            is_director: d.is_director,
            is_manager: d.is_manager,
            is_instructor: d.is_instructor,
            checked: false,
            index: i,
          });
        });

        directories.users = directories.users;
      })
      .catch((err) => {
        console.error(err);
      });
  }

  async function sendServerData() {
    let data = {
      dt: `${dateOperation}T00:00:00+03:00`,
      employees: directories.users.filter((u) => u.checked).map((u) => u.id),
      amount: amount ? amount : 0,
      note: description,
      source_id: 1,
    };

    fetch("/api/penalty", {
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
    dateOperation = formatDate(new Date());

    tg = window.Telegram.WebApp;

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
  <div class="w-full px-3 pb-64">
    <div class="m-1">
      <h2 class="ml-3 text-3xl font-bold">Штрафы</h2>
    </div>

    <div class="m-1 text-2xl">
      <div class="mt-2 grid grid-cols-1 gap-1.5">
        <!-- Дата операции -->
        <label class="block">
          <span class="ml-1 text-gray-700">Дата</span>
          <input
            bind:value={dateOperation}
            type="date"
            class="mt-1 block h-10 w-full rounded border-gray-300 py-1 text-xl shadow-sm focus:border-gray-500 focus:ring focus:ring-gray-400 focus:ring-opacity-50"
          />
        </label>

        <!-- Сотрудники -->
        <label class="block">
          <span class="text-gray-700">Сотрудники</span>
          <div class="relative">
            <button
              on:click={() => (visibleDropdownList = !visibleDropdownList)}
              class="mt-1 flex h-10 w-full place-content-between items-center rounded border border-gray-300 bg-white py-1 text-xl shadow-sm focus:border-gray-500 focus:ring focus:ring-gray-400 focus:ring-opacity-50"
            >
              <span class="ml-2 leading-none"
                >{directories && directories.users
                  ? directories.users.filter((u) => u.checked).length
                    ? directories.users
                        .filter((u) => u.checked)
                        .map((u) => u.name)
                        .join(", ")
                    : ""
                  : ""}</span
              >
              <svg class="ml-2 mr-2 h-6 w-6 text-gray-500" viewBox="0 0 256 256">
                <path
                  fill="currentColor"
                  d="m216.49 104.49l-80 80a12 12 0 0 1-17 0l-80-80a12 12 0 0 1 17-17L128 159l71.51-71.52a12 12 0 0 1 17 17Z"
                />
              </svg>
            </button>
            {#if visibleDropdownList}
              <div class="absolute mb-10 flex w-full flex-col rounded border border-gray-400 bg-white text-xl shadow-lg">
                {#if directories && directories.users}
                  {#each directories.users.filter((u) => u.is_director || u.is_manager || u.is_instructor) as employee, index}
                    <div class="block">
                      <div class="px-2 py-1">
                        <label class="inline-flex items-center">
                          <input
                            bind:checked={directories.users[employee.index].checked}
                            type="checkbox"
                            class="h-5 w-5 rounded border-gray-300 text-xl shadow-sm focus:border-gray-500 focus:ring focus:ring-gray-400 focus:ring-opacity-50 focus:ring-offset-0"
                          />
                          <span class="ml-2">{directories.users[employee.index].name}</span>
                        </label>
                      </div>
                    </div>
                  {/each}
                {/if}
              </div>
            {/if}
          </div>
        </label>

        <!-- Сумма -->
        <label class="block">
          <span class="ml-1 text-gray-700">Размер штрафа (каждому)</span>
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
