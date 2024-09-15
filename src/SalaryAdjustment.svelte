<script>
  import { onMount } from "svelte";

  const formatter = new Intl.NumberFormat("ru");
  const SECRET =
    "n5v0k1DkLVuLzGR2uftfh9bR1woL8PL6RlHeJBtLEk0lsA6KsiGYwRL3PvKOb4yCXZVPtjEsDeoipRV9s3sJ4q21AEpWwbo5ncixVKqz6iW0kLIjq035M5etxu0xEZnd";

  $: tg = null;

  const months = [
    "январь",
    "февраль",
    "март",
    "апрель",
    "май",
    "июнь",
    "июль",
    "август",
    "сентябрь",
    "октябрь",
    "ноябрь",
    "декабрь",
  ];
  const nameMonths = [
    "января",
    "февраля",
    "марта",
    "апреля",
    "мая",
    "июня",
    "июля",
    "августа",
    "сентября",
    "октября",
    "ноября",
    "декабря",
  ];

  let years = [2024];

  let username = "";
  let selectedYear = 2024;
  let selectedMonth;
  let selectedEmployee = "";
  let salaryDays = [];
  let employees = [];

  let tempSalary = 0;
  let tempPercent = 0;
  let tempPenalty = 0;

  $: tempSalary = Number(tempSalary.toString().replaceAll(/[^0-9./]/gi, ""));
  $: tempPercent = Number(tempPercent.toString().replaceAll(/[^0-9./]/gi, ""));
  $: tempPenalty = Number(tempPenalty.toString().replaceAll(/[^0-9./]/gi, ""));

  $: filteredSalaries(selectedYear, selectedMonth, selectedEmployee);

  const currentDate = new Date();

  // Видимость кнопки сохранения
  $: tg && tg.MainButton
    ? (tg.MainButton.isVisible = salaryDays.some(
        (p) => p.salary != p.salaryNew || p.percent != p.percentNew || p.penalty != p.penaltyNew
      ))
    : null;

  async function getServerSalary() {
    fetch("/api/user_salary", {
      method: "POST",
      headers: { "Content-Type": "application/json;charset=utf-8" },
      body: JSON.stringify({
        key: SECRET,
        username: "all-users",
      }),
    })
      .then((response) => {
        return response.json();
      })
      .then((data) => {
        data.salary_days.forEach((sd) => {
          salaryDays.push({
            period: `${sd.day} ${nameMonths[sd.month]}`,
            year: sd.year,
            month: sd.month,
            day: sd.day,
            userID: sd.user_id,
            userName: sd.user_name,
            salary: sd.salary,
            salaryNew: sd.salary,
            percent: sd.percent,
            percentNew: sd.percent,
            penalty: sd.penalty,
            penaltyNew: sd.penalty,
            editSalary: false,
            editPercent: false,
            editPenalty: false,
            insert: false,
          });
        });
        employees = data.employees.sort((a, b) => {
          if (a.name > b.name) return 1;
          if (a.name < b.name) return -1;
          return 0;
        });
        years = data.years;

        const currentDate = new Date();
        selectedYear = currentDate.getFullYear();
        selectedMonth = currentDate.getMonth();
      })
      .catch((err) => {
        console.error(err);
      });
  }

  async function sendServerData() {
    let data = [];

    salaryDays.forEach((p) => {
      if (p.salary != p.salaryNew || p.percent != p.percentNew || p.penalty != p.penaltyNew) {
        data.push({
          year: p.year,
          month: p.month + 1,
          day: p.day,
          user_id: p.userID,
          salary: p.salaryNew,
          percent: p.percentNew,
          penalty: p.penaltyNew,
          insert: p.insert,
        });
      }
    });

    fetch("/api/edit_salary", {
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

  function getMaxDay(year, month) {
    const cd = new Date();
    return cd.getFullYear() == year && cd.getMonth() == month ? cd.getDate() : new Date(year, month + 1, 0).getDate();
  }

  function filteredSalaries(year, month, employeeID) {
    if (employeeID == 0) return [];

    const maxDate = getMaxDay(year, month);
    let monthDays = {};
    for (let d = 0; d < maxDate; d++) {
      monthDays[d + 1] = true;
    }

    let fs = [];
    salaryDays.forEach((d) => {
      if (d.year == year && d.month == month && d.userID == employeeID) {
        fs.push(d);
        monthDays[d.day] = false;
      }
    });

    Object.entries(monthDays).forEach((d) => {
      if (d[1]) {
        salaryDays.push({
          period: `${d[0]} ${nameMonths[month]}`,
          year: year,
          month: month,
          day: Number(d[0]),
          userID: employeeID,
          userName: "",
          salary: 0,
          salaryNew: 0,
          percent: 0,
          percentNew: 0,
          penalty: 0,
          penaltyNew: 0,
          editSalary: false,
          editPercent: false,
          editPenalty: false,
          insert: true,
        });
      }
    });

    salaryDays.sort((a, b) => {
      if (a.year - b.year != 0) return a.year - b.year;
      if (a.month - b.month != 0) return a.month - b.month;
      return a.day - b.day;
    });
  }

  // Загружаем данные при создании формы
  onMount(() => {
    tg = window.Telegram.WebApp;
    if (tg && tg.initDataUnsafe && tg.initDataUnsafe.user) {
      username = tg.initDataUnsafe.user.username;
    }

    getServerSalary();

    if (tg && tg.MainButton) {
      tg.MainButton.setText("Сохранить все изменения");
      tg.MainButton.color = "#a3e635";
      tg.backgroundColor = "#e5e7eb";
      tg.headerColor = "#e5e7eb";
    }

    tg.onEvent("mainButtonClicked", () => sendServerData());
    tg.enableClosingConfirmation();
  });

  function focus(node) {
    node.focus();

    return {
      destroy() {},
    };
  }
</script>

<main class="max-w-lg bg-gray-200">
  <div class="w-full px-2 pb-72">
    <h2 class="text-2xl font-bold mb-2">Корректировка ЗП</h2>

    <div class="grid grid-cols-1 gap-1.5">
      <div class="grid grid-cols-3 grid-rows-2">
        <span class="text-gray-700">Сотрудник</span>
        <span class="text-gray-700">Год</span>
        <span class="text-gray-700">Месяц</span>
        <select
          bind:value={selectedEmployee}
          class="w-full rounded border-gray-300 px-2 py-0.5 text-sm shadow-sm focus:border-indigo-300 focus:ring focus:ring-indigo-200 focus:ring-opacity-50"
        >
          {#if employees.length}
            {#each employees as e}
              <option value={e.id}>{e.name}</option>
            {/each}
          {/if}
        </select>
        <select
          bind:value={selectedYear}
          class="w-full rounded border-gray-300 px-2 py-0.5 text-sm shadow-sm focus:border-indigo-300 focus:ring focus:ring-indigo-200 focus:ring-opacity-50"
        >
          {#if years.length}
            {#each years as year}
              <option value={year} class={year == currentDate.getFullYear() ? "selected" : ""}>{year}</option>
            {/each}
          {/if}
        </select>
        <select
          bind:value={selectedMonth}
          class="w-full text-sm rounded border-gray-300 px-2 py-0.5 shadow-sm focus:border-indigo-300 focus:ring focus:ring-indigo-200 focus:ring-opacity-50"
        >
          {#if months.length}
            {#each months as month, index}
              <option value={index} class={index == currentDate.getMonth() ? "selected" : ""}>{month}</option>
            {/each}
          {/if}
        </select>
      </div>
    </div>

    <div class="mt-1 grid grid-cols-4 text-sm">
      <div class="col-span-4">Детализация</div>

      <div class="border-b border-r border-gray-200 bg-slate-300 py-0.5 text-center font-bold">Дата</div>
      <div class="border-b border-r border-gray-200 bg-slate-300 py-0.5 text-center font-bold">Оклад</div>
      <div class="border-b border-r border-gray-200 bg-slate-300 py-0.5 text-center font-bold">Процент</div>
      <div class="border-b border-gray-200 bg-slate-300 py-0.5 text-center font-bold">Штраф</div>

      {#if salaryDays.length}
        {#each salaryDays.filter((d) => d.year == selectedYear && d.month == selectedMonth && d.userID == selectedEmployee) as p}
          <div class="border-b border-r border-gray-300 px-1">{p.period}</div>

          {#if p.editSalary}
            <input
              on:keydown={(e) => {
                if (e.key == "Enter") {
                  if (p.salaryNew != tempSalary) {
                    p.salaryNew = tempSalary;
                  }
                  p.editSalary = false;
                }
              }}
              on:keydown={(e) => (e.key == "Escape" ? (p.editSalary = false) : "")}
              on:focusout={() => (p.editSalary = false)}
              class="border-b border-r border-gray-300 px-1 text-right"
              type=""
              bind:value={tempSalary}
              use:focus
            />
          {:else}
            <div
              on:click={() => {
                p.editSalary = !p.editSalary;
                tempSalary = p.salaryNew;
              }}
              class="border-b border-r border-gray-300 px-1 text-right {p.salary != p.salaryNew
                ? 'bg-green-300'
                : 'text-gray-400'}"
            >
              {formatter.format(p.salaryNew)}
            </div>
          {/if}

          {#if p.editPercent}
            <input
              on:keydown={(e) => {
                if (e.key == "Enter") {
                  if (p.percentNew != tempPercent) {
                    p.percentNew = tempPercent;
                  }
                  p.editPercent = false;
                }
              }}
              on:keydown={(e) => (e.key == "Escape" ? (p.editPercent = false) : "")}
              on:focusout={() => (p.editPercent = false)}
              class="border-b border-r border-gray-300 px-1 text-right"
              type=""
              bind:value={tempPercent}
              use:focus
            />
          {:else}
            <div
              on:click={() => {
                p.editPercent = !p.editPercent;
                tempPercent = p.percentNew;
              }}
              class="border-b border-r border-gray-300 px-1 text-right {p.percent != p.percentNew
                ? 'bg-green-300'
                : 'text-gray-400'}"
            >
              {formatter.format(p.percentNew)}
            </div>
          {/if}

          {#if p.editPenalty}
            <input
              on:keydown={(e) => {
                if (e.key == "Enter") {
                  if (p.penaltyNew != tempPenalty) {
                    p.penaltyNew = tempPenalty;
                  }
                  p.editPenalty = false;
                }
              }}
              on:keydown={(e) => (e.key == "Escape" ? (p.editPenalty = false) : "")}
              on:focusout={() => (p.editPenalty = false)}
              class="border-b border-r border-gray-300 px-1 text-right"
              type=""
              bind:value={tempPenalty}
              use:focus
            />
          {:else}
            <div
              on:click={() => {
                p.editPenalty = !p.editPenalty;
                tempPenalty = p.penaltyNew;
              }}
              class="border-b border-r border-gray-300 px-1 text-right {p.penalty != p.penaltyNew
                ? 'bg-green-300'
                : 'text-gray-400'}"
            >
              {formatter.format(p.penaltyNew)}
            </div>
          {/if}
        {/each}
      {/if}
    </div>
  </div>
</main>
