<script setup>
    import { ref, computed } from 'vue';
    import { solar2lunar } from 'solarLunar';
    import domtoimage from 'dom-to-image';
    import { saveAs } from 'file-saver';

    const getDateInfo = (dateObj) => {
        if (!dateObj instanceof Date) return;
        const year = dateObj.getFullYear(),
            month = dateObj.getMonth() + 1,
            date = dateObj.getDate(),
            day = dateObj.getDay();
        // 获取农历信息, dayCn: 中文的日 (如: 初一); monthCn: 中文的月 (如: 一月); isTerm: 是否有节气; term: 节气 (如: 立春)
        const { dayCn, monthCn, isTerm, term } = solar2lunar(year, month, date);
        // 如果有节气则显示节气, 否则显示日期, 日期若为初一则替换为月份.
        const lunar = isTerm ? term : (dayCn == '初一' ? monthCn : dayCn);
        return {
            year,
            month,
            date,
            day,
            lunar
        }
    }
    const getMonthInfo = (year, month) => {
        // 当前年月的中英文
        const yearCnText = (year + '').split('').map(n => ['零', '一', '二', '三', '四', '五', '六', '七', '八', '九'][n]).join(''),
            monthCnText = ['一', '二', '三', '四', '五', '六', '七', '八', '九', '十', '十一', '十二'][month - 1] + '月',
            monthEnText = ['January', 'February', 'March', 'April', 'May', 'June', 'July', 'August', 'September', 'October', 'November', 'December'][month - 1];
        // 当前月的天数 (或: 该月最后一天)
        // new Date 的 "日" < 1 时自动从后往前取上个月的日期
        // "(month - 1 + 1)": 减1是因为传入月份应从零开始 (即: 传入0代表一月, 1代表二月); 加一是因为要从下个月往回推
        const monthDates = (new Date(year, (month - 1 + 1), 0)).getDate();
        // 当前月的日历信息 (月数组)
        /*
        [
            (这个月的第一周) [ { (周一) },                { (周二 (上个月最后一天)) }, { (周三 (这个月一号)) }, (...), { (周日) } ],
            (这个月的第二周) [ { (周一) },                { (周二) },                 { (周三) },             (...), { (周日) } ],
            (...),
            (这个月的第五周) [ { (周一 (这个月最后一天)) }, { (周二 (下个月一号)) },    { (周三) },              (...), { (周日) } ],
        ]
        */
        let dates = [];
        // 获取当前月的第一周
        let firstWeekOfThisMonth = [];
        // 也就是当前月第一天所在的星期
        // 当前月1号是周几, 一周从周一开始 (而不是周日) 所以要把0改为7然后再减一
        let firstDateDayOfThisMonth = (new Date(year, month - 1, 1)).getDay();
        if (firstDateDayOfThisMonth == 0) firstDateDayOfThisMonth = 7;
        firstDateDayOfThisMonth--;
        // 获取上个月的最后n天
        for (let n = 0; n > (firstDateDayOfThisMonth * -1); n--)
            firstWeekOfThisMonth.push({ ...getDateInfo(new Date(year, month - 1, n)), notThisMonth: 1 });
        firstWeekOfThisMonth.reverse();
        // 加上这个月的第一周剩下的天数
        for (let n = 1; n <= (7 - firstDateDayOfThisMonth); n++) firstWeekOfThisMonth.push(getDateInfo(new Date(year, month - 1, n)));
        // 加到月数组里
        dates.push(firstWeekOfThisMonth);
        // 当前月剩下的周
        while (1) {
            // 结束标志 (即: 此标志后的date均非本月的date, 同时生成完该周数据就结束生成)
            let isBreak = 0;
            // 取上一周最后一天的日期 (date)
            let lastDate = dates.at(-1).at(-1).date;
            // 本周第一天的日期 (date)
            let startDate = lastDate + 1;
            // 本周
            let thisWeek = [];
            // 生成
            for (let d = startDate; d < startDate + 7; d++) {
                let dateInfo = getDateInfo(new Date(year, month - 1, d));
                thisWeek.push(isBreak ? { ...dateInfo, notThisMonth: 1 } : dateInfo);
                if (dateInfo.date == monthDates) isBreak = 1;
            }
            // 添加到月数组里
            dates.push(thisWeek);
            // 判断是否生成完毕
            if (isBreak) break;
        }
        return {
            year,
            yearCnText,
            month,
            monthCnText,
            monthEnText,
            monthDates,
            dates
        }
    }

    window.getMonthInfo = getMonthInfo;

    let year = 2023;
    let yearInfos = []
    for (let y = 1; y <= 12; y++) yearInfos.push(getMonthInfo(year, y));

    let currentMonth = ref(1);
    let currentMonthData = computed(() => yearInfos[currentMonth.value - 1]);


    const calendarDom = ref(null);
    const previousMonth = () => currentMonth.value = currentMonth.value == 1 ? 12 : (currentMonth.value - 1);
    const nextMonth = () => currentMonth.value = currentMonth.value == 12 ? 1 : (currentMonth.value + 1);
    const downloadImage = () => {
        domtoimage.toBlob(calendarDom.value).then(blob => {
            saveAs(blob, currentMonth.value + '.png');
        });
    }
</script>

<template>
    <div class="calendar" ref="calendarDom">
        <div class="title">
            <div class="left"><div class="month">{{ currentMonthData.monthCnText }}</div></div>
            <div class="right">
                <div class="year">{{ year }}</div>
                <div class="month-en">{{ currentMonthData.monthEnText }}</div>
            </div>
        </div>
        <div class="table">
            <table class="calendar-table">
                <thead>
                    <tr>
                        <th>周一</th>
                        <th>周二</th>
                        <th>周三</th>
                        <th>周四</th>
                        <th>周五</th>
                        <th>周六</th>
                        <th>周日</th>
                    </tr>
                </thead>
                <tbody>
                    <tr v-for="week in currentMonthData.dates">
                        <td v-for="date in week">
                            <p class="date" :class="date.notThisMonth ? 'not-this-month' : ''">{{ date.date }}</p>
                            <p class="lunar">{{ date.lunar }}</p>
                        </td>
                    </tr>
                </tbody>
            </table>
        </div>
    </div>
    <div class="controls">
        <button @click="previousMonth()">上个月</button>
        <button @click="nextMonth()">下个月</button>
        <button @click="downloadImage()">下载图片</button>
    </div>
</template>

<style lang="less" scoped>
    .calendar {
        width: 333px;
        height: 500px;
        background-color: white;
        overflow: auto;
        padding: 100px 20px 20px 20px;
        border: 1px solid black;

        .title {
            display: flex;

            .left {
                .month {
                    font-size: 3em;
                }
            }
            .right {
                display: flex;
                flex-direction: column;
                justify-content: space-evenly;
                font-family: '黑体';
                text-align: center;
                font-size: 1.1em;

                .year {
                    background-color: black;
                    border-radius: 5px;
                    color: white;
                    padding: 0 1em;
                }
                .month-en {}
            }
        }
        .table {
            .calendar-table {
                width: 100%;

                tr {
                    th {
                        font-size: 1.1em;
                    }
                    td {
                        .lunar, .date {
                            margin: 0;
                            text-align: center;
                        }
                        .date {
                            font-family: '黑体';
                            font-size: 1.4em;
                            &:not(.not-this-month) {
                                font-weight: 900;
                            }
                        }
                        .lunar {
                            // font-size: 0.9em;
                        }
                    }
                }
            }
        }
    }
    .controls {
        margin-top: 50px;
    }
</style>

<style>
    body {
        background-color: yellow;
        padding: 59px;
    }
    * {
        box-sizing: border-box;
    }
</style>
