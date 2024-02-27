<script setup lang="ts">
import * as echarts from "echarts";
import { GbfrActWs } from "gbfr-act-ws-package";
import { onMounted, ref } from "vue";

const echartsDom = ref();

const params = new URLSearchParams(window.location.search);
const valTrue = ["true", "1"];
const showTitle = valTrue.includes(params.get("title") || "true");
const theme = params.get("theme") || "dark";

const rgb = (r: number, g: number, b: number) => `rgb(${r}, ${g}, ${b})`;

const actorConfigMap = new Map([
  ["9498420d", { name: "姬塔", color: rgb(117, 117, 117) }],
  ["26a4848a", { name: "古兰", color: rgb(117, 117, 117) }],
  ["c3155079", { name: "塞达", color: rgb(46, 125, 50) }],
  ["34d4fd8f", { name: "卡塔莉娜", color: rgb(63, 81, 181) }],
  ["f8d73d33", { name: "拉卡姆", color: rgb(158, 157, 36) }],
  ["7b5934ad", { name: "伊欧", color: rgb(126, 87, 194) }],
  ["443d46bb", { name: "欧根", color: rgb(153, 23, 23) }],
  ["a9d6569e", { name: "萝赛塔", color: rgb(121, 85, 72) }],
  ["2b4aa114", { name: "夏洛特", color: rgb(254, 179, 0) }],
  ["bcc238de", { name: "冈达葛萨", color: rgb(136, 14, 79) }],
  ["fba6615d", { name: "菲莉", color: rgb(121, 134, 203) }],
  ["601aa977", { name: "娜露梅", color: rgb(244, 143, 177) }],
  ["63a7c3f0", { name: "兰斯洛特", color: rgb(0, 151, 167) }],
  ["f96a90c2", { name: "巴恩", color: rgb(255, 152, 0) }],
  ["28ac1108", { name: "珀西瓦尔", color: rgb(233, 30, 99) }],
  ["94e2514e", { name: "齐格飞", color: rgb(0, 185, 247) }],
  ["6fdd6932", { name: "卡莉奥丝特罗", color: rgb(255, 202, 40) }],
  ["c97f3365", { name: "尤达拉哈", color: rgb(79, 195, 247) }],
  ["d16cfbde", { name: "巴萨拉卡", color: rgb(78, 52, 46) }],
  ["8056abcd", { name: "伊德", color: rgb(21, 28, 100) }],
  ["121d9d67", { name: "奥义连锁", color: "#444" }],
]);

const style = {
  bleed: 8,
  names: {
    color: theme === "dark" ? "white" : "black",
    fontSize: "12px",
    fontFamily: "Microsoft YaHei",
    margin: 8,
  },
  damages: {
    color: theme === "dark" ? undefined : "gray",
    fontSize: "12px",
    fontFamily: "Microsoft YaHei",
    distance: 5,
  },
};

onMounted(() => {
  const gbfrActWs = new GbfrActWs({
    port: 24399,
    updateInterval: 500,
    reconnectTimeout: 3000,
    maxCombats: 10,
  });

  const chartBox = echarts.init(
    echartsDom.value,
    theme === "dark" ? "dark" : null,
    {
      renderer: "svg",
      useDirtyRect: true,
      locale: "ZH",
    },
  );

  const option: echarts.EChartsOption = {
    backgroundColor: theme === "dark" ? "rgba(0,0,0,0.5)" : undefined,
    grid: {
      top: showTitle ? 25 : 5,
      bottom: 5,
    },
    xAxis: {
      max: "dataMax",
      axisLine: {
        show: false,
      },
      splitLine: {
        show: false,
      },
      axisLabel: {
        show: false,
      },
    },
    yAxis: {
      position: "left",
      type: "category",
      inverse: true,
      axisLine: {
        show: false,
      },
      axisTick: {
        show: false,
      },
      axisLabel: {
        show: true,
        fontWeight: "normal",
        overflow: "break",
        color: style.names.color,
        fontSize: parseInt(style.names.fontSize),
        fontFamily: style.names.fontFamily,
        margin: style.names.margin,
      },
      animationDuration: 0,
      animationDurationUpdate: 300,
    },
    series: [
      {
        realtimeSort: true,
        type: "bar",
        label: {
          show: true,
          position: "right",
          valueAnimation: true,
          formatter: (params: any): string => params.value.toLocaleString(),
          textBorderWidth: 1,
          textBorderType: "solid",
          color: style.damages.color,
          fontSize: parseInt(style.damages.fontSize),
          fontFamily: style.damages.fontFamily,
          distance: style.damages.distance,
        },
      },
    ],
    legend: {
      show: true,
    },
    animationDuration: 0,
    animationDurationUpdate: 300,
    animationEasing: "linear",
    animationEasingUpdate: "linear",
  };
  chartBox.setOption(option);

  let lastActorNames = "";

  const getTextSize = (text: string, font: { fontSize: string; fontFamily: string }) => {
    const span = document.getElementById("textSize");
    if (!span) return { width: 0, height: 0 };
    span.style.fontSize = font.fontSize;
    span.style.fontFamily = font.fontFamily;
    span.textContent = text;
    return { width: span.offsetWidth, height: span.offsetHeight };
  };

  gbfrActWs.on("combat_data", (data): void => {
    // update yAxis
    if (lastActorNames !== data.actors.join("/")) {
      const names = data.actors.map((v) => {
        let name = actorConfigMap.get(v.hexId)?.name ?? v.hexId;
        if (
          data.actors.find(
            (d) => d.partyIdx !== v.partyIdx && d.hexId === v.hexId,
          )
        ) {
          const index = `${v.partyIdx + 1}`;
          return `[${index}]${name}`;
        }
        return name;
      });
      const left = style.names.margin + style.bleed
        + names.reduce((max, name) => Math.max(getTextSize(name, style.names).width, max), 0);
      chartBox.setOption<echarts.EChartsOption>({ grid: { left }, yAxis: { data: names } });
    }

    lastActorNames = data.actors.join("/");

    const right = style.damages.distance + style.bleed
      + getTextSize(data.actors.reduce((max, actor) => Math.max(max, actor.damage), 0).toLocaleString(), style.damages)
        .width;

    chartBox.setOption<echarts.EChartsOption>({
      grid: { right },
      title: {
        show: showTitle,
        text: `${data.duration.MMSS} - 小队秒伤：${
          Math.round(
            data.partyDamage / data.duration.seconds,
          ).toLocaleString()
        }`,
      },
      series: [
        {
          data: data.actors.map((v) => ({
            value: v.damage,
          })),
          label: {
            show: true,
          },
          itemStyle: {
            color: (params: any) => {
              try {
                const hexId = data.actors[params!.dataIndex].hexId;
                return actorConfigMap.get(hexId)?.color ?? "";
              } catch {
                return "";
              }
            },
          },
        },
      ],
    });
  });
  window.addEventListener("resize", function() {
    chartBox.resize();
  });
});
</script>

<template>
  <div ref="echartsDom" class="echarts"></div>
  <span id="textSize"></span>
</template>

<style scoped>
.echarts {
  width: 100vw;
  height: 100vh;
}

#textSize {
  visibility: hidden;
  white-space: nowrap;
  position: absolute;
  top: -9999px;
}
</style>
