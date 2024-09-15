<script>
  import { onMount } from "svelte";

  const formatter = new Intl.NumberFormat("ru", { maximumFractionDigits: 0 });
  const SECRET =
    "n5v0k1DkLVuLzGR2uftfh9bR1woL8PL6RlHeJBtLEk0lsA6KsiGYwRL3PvKOb4yCXZVPtjEsDeoipRV9s3sJ4q21AEpWwbo5ncixVKqz6iW0kLIjq035M5etxu0xEZnd";

  $: tg = null;

  const months = ["январь", "февраль", "март", "апрель", "май", "июнь", "июль", "август", "сентябрь", "октябрь", "ноябрь", "декабрь"];
  const nameMonths = ["января", "февраля", "марта", "апреля", "мая", "июня", "июля", "августа", "сентября", "октября", "ноября", "декабря"];

  let years = [2024];
  let directories = {};

  let username = "";
  let tgInvestor = "";
  let selectedYear = 2024;
  let selectedMonth;
  let investor = {};
  let investorDays = [];
  let typeTechniques = {};
  let filteredInvestor = [];

  let summury = {
    fullIncome: 0,
    income: 0,
    expense: 0,
    dividend: 0,
    fuel: 0,
    fullRepair: 0,
    repair: 0,
    fullSpare: 0,
    spare: 0,
    specificGravity: 0,
    profit: 0,
  };

  $: calc(selectedYear, selectedMonth);
  $: visibleRaschet = true;
  $: visibleDetail = false;

  const currentDate = new Date();

  function calc(year, month) {
    summury = {
      fullIncome: 0,
      income: 0,
      expense: 0,
      dividend: 0,
      fuel: 0,
      fullRepair: 0,
      repair: 0,
      fullSpare: 0,
      spare: 0,
      specificGravity: 0,
      profit: 0,
    };
    filteredInvestor = [];
    investorDays.forEach((d) => {
      if (d.year == year && d.month == month) {
        summury.fullIncome += d.full_income;
        summury.income += d.income;
        summury.expense += d.expense;
        summury.dividend += d.dividend;
        summury.fuel += d.fuel;
        summury.fullRepair += d.full_repair;
        summury.repair += d.repair;
        summury.fullSpare += d.full_spare;
        summury.spare += d.spare;

        filteredInvestor.push({
          period: `${d.day} ${nameMonths[d.month]}`,
          fullIncome: d.full_income,
          income: d.income,
          expense: d.expense,
          dividend: d.dividend,
          fuel: d.fuel,
          fullRepair: d.full_repair,
          repair: d.repair,
          fullSpare: d.full_spare,
          spare: d.spare,
        });
      }
    });

    summury.specificGravity = summury.income / summury.fullIncome;
    summury.profit =
      summury.income -
      (summury.fuel + summury.expense + summury.fullRepair + summury.fullSpare) * summury.specificGravity -
      summury.repair -
      summury.spare;

    filteredInvestor = filteredInvestor;
  }

  async function getServerInvestorData(nik) {
    fetch("/api/investor", {
      method: "POST",
      headers: { "Content-Type": "application/json;charset=utf-8" },
      body: JSON.stringify({
        key: SECRET,
        username: nik,
      }),
    })
      .then((response) => {
        return response.json();
      })
      .then((data) => {
        investorDays = data.report.sort((a, b) => {
          if (a.year - b.year != 0) return a.year - b.year;
          if (a.month - b.month != 0) return a.month - b.month;
          return a.day - b.day;
        });
        typeTechniques = data.techs;
        investor = data.investor;
        years = data.years;

        const currentDate = new Date();
        selectedYear = currentDate.getFullYear();
        selectedMonth = currentDate.getMonth();
      })
      .catch((err) => {
        console.error(err);
      });
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

  $: getServerInvestorData(tgInvestor);

  // Загружаем данные при создании формы
  onMount(() => {
    tg = window.Telegram.WebApp;
    if (tg && tg.initDataUnsafe && tg.initDataUnsafe.user) {
      username = tg.initDataUnsafe.user.username;
      getServerInvestorData(username);
    }

    getServerDirectories();

    if (tg && tg.MainButton) {
      tg.MainButton.setText("Закрыть");
      tg.MainButton.color = "#a3e635";
      tg.backgroundColor = "#e5e7eb";
      tg.headerColor = "#e5e7eb";
    }
  });
</script>

<main class="max-w-lg bg-gray-200">
  <div class="w-full px-2 pb-64">
    <div class="grid grid-cols-1 gap-1.5">
      <div class="grid grid-cols-3 grid-rows-2">
        {#if username == "axm9t"}
          <span class="text-gray-700">Инвестор</span>
        {:else}
          <span class="row-span-2 text-gray-700">
            <img class=" w-16 m-2 pt-3" src="extrim.png" alt="ЭКСТРИМ АРХЫЗ" />
          </span>
        {/if}
        <span class="text-gray-700">Год</span>
        <span class="text-gray-700">Месяц</span>
        {#if username == "axm9t"}
          <select
            bind:value={tgInvestor}
            class="w-full text-sm rounded border-gray-300 px-2 py-0.5 shadow-sm focus:border-indigo-300 focus:ring focus:ring-indigo-200 focus:ring-opacity-50"
          >
            {#if directories && directories.users}
              {#each directories.users as inv}
                {#if inv.is_investor}
                  <option value={inv.tg}>{inv.name}</option>
                {/if}
              {/each}
            {/if}
          </select>
        {/if}
        <select
          bind:value={selectedYear}
          class="w-full text-sm rounded border-gray-300 px-2 py-0.5 shadow-sm focus:border-indigo-300 focus:ring focus:ring-indigo-200 focus:ring-opacity-50"
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

    <div class="mt-1 flex justify-stretch gap-1 border-b-4 border-t-4 border-sky-400">
      <div class="my-1 flex w-full flex-col text-xl">
        <div class="flex text-2xl">
          <span class="grow">К выплате</span>
          <span class="w-36 shrink text-right"
            >{summury.profit && investor.percent ? formatter.format(summury.profit * investor.percent) : 0}</span
          >
        </div>
        <div class="flex">
          <span class="grow">Условия</span>
          <span class="w-36 shrink text-right">{investor.percent ? formatter.format(investor.percent * 100) : 0}%</span>
        </div>
        <div class="flex">
          <span class="grow">Выплачено</span>
          <span class="w-36 shrink text-right">{summury.dividend ? formatter.format(summury.dividend) : 0}</span>
        </div>
        <div class="flex">
          <span class="grow">Осталось</span>
          <span class="w-36 shrink text-right"
            >{summury.profit && investor.percent
              ? formatter.format(summury.profit * investor.percent - summury.dividend)
              : 0 - summury.dividend}</span
          >
        </div>
      </div>
    </div>

    <div class="m-1 block">
      <nav class="grid grid-cols-2 gap-6 text-xl">
        <button
          on:click={() => {
            visibleRaschet = true;
            visibleDetail = false;
          }}
          class="grow rounded-lg {visibleRaschet ? 'bg-gray-300' : ''} p-1 text-center font-medium hover:bg-gray-300">Расчеты</button
        >
        <button
          on:click={() => {
            visibleRaschet = false;
            visibleDetail = true;
          }}
          class="grow rounded-lg {visibleDetail ? 'bg-gray-300' : ''} p-1 text-center font-medium hover:bg-gray-300"
          aria-current="page">Детализация</button
        >
      </nav>
    </div>

    {#if visibleRaschet}
      <div class="mt-1 flex justify-stretch gap-1">
        <div class="my-1 flex w-full flex-col text-xl">
          <div class="flex">
            <span class="grow">Прибыль</span>
            <span class="w-36 shrink text-right">{formatter.format(summury.income)}</span>
          </div>
          <div class="flex">
            <span class="grow">Бензин</span>
            <span class="w-36 shrink text-right"
              >{investor.percent ? formatter.format(summury.fuel * summury.specificGravity * investor.percent) : 0}</span
            >
          </div>
          <div class="flex">
            <span class="grow">Ремонт</span>
            <span class="w-36 shrink text-right"
              >{investor.percent
                ? formatter.format(summury.fullRepair * summury.specificGravity * investor.percent + summury.repair)
                : 0}</span
            >
          </div>
          <div class="flex">
            <span class="grow">Запчасти</span>
            <span class="w-36 shrink text-right"
              >{investor.percent
                ? formatter.format(summury.fullSpare * summury.specificGravity * investor.percent + summury.spare)
                : 0}</span
            >
          </div>
          <div class="flex">
            <span class="grow">Расходы</span>
            <span class="w-36 shrink text-right"
              >{investor.percent ? formatter.format(summury.expense * summury.specificGravity * investor.percent) : 0}</span
            >
          </div>
          <div class="flex">
            <span class="grow">Чистая прибыль</span>
            <span class="w-36 shrink text-right">{investor.percent ? formatter.format(summury.profit * investor.percent) : 0}</span>
          </div>
        </div>
      </div>
    {/if}

    {#if visibleDetail}
      <div class="mt-1 grid grid-cols-4 text-sm">
        <div class="row-span-2 border-b border-r border-gray-200 bg-slate-300 pt-3 center text-center font-bold">Дата</div>
        <div class="col-span-3 border-b border-r border-gray-200 bg-slate-300 py-0.5 text-center font-bold">Прибыль</div>

        <div class="border-b border-r border-gray-200 bg-slate-300 py-0.5 text-center font-bold">Прибыль</div>
        <div class="border-b border-r border-gray-200 bg-slate-300 py-0.5 text-center font-bold">Бензин</div>
        <div class="border-b border-r border-gray-200 bg-slate-300 py-0.5 text-center font-bold">Ремонт</div>

        <div class="border-r border-r-gray-200 bg-gray-300 px-1 py-0.5 text-right font-bold"></div>
        <div class="col-span-3 border-r border-gray-200 bg-gray-300 px-1 py-0.5 text-right font-bold">
          Инд. ремонт: {formatter.format(summury.repair)}
        </div>

        <div class="border-r border-r-gray-200 bg-gray-300 px-1 py-0.5 text-right font-bold">Итого:</div>
        <div class="col-span-3 border-r border-gray-200 bg-gray-300 px-1 py-0.5 text-right font-bold">
          Общий ремонт: {investor.percent ? formatter.format(summury.fullRepair * investor.percent) : 0}
        </div>

        {#if filteredInvestor.length}
          {#each filteredInvestor as inv}
            <div class="border-b border-r border-gray-300 px-1">{inv.period}</div>
            <div class="border-b border-r border-gray-300 px-1 text-right">{formatter.format(inv.income)}</div>
            <div class="border-b border-r border-gray-300 px-1 text-right">
              {investor.percent ? formatter.format(inv.fuel * investor.percent) : 0}
            </div>
            <div class="border-b border-gray-300 px-1 text-right">
              {investor.percent ? formatter.format(inv.fullRepair * investor.percent + inv.repair) : 0}
            </div>
          {/each}
        {/if}
      </div>
    {/if}
  </div>
</main>
