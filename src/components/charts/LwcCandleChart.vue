<script setup lang="ts">
import {
  CandlestickSeries,
  HistogramSeries,
  LineSeries,
  AreaSeries,
  createChart,
  createSeriesMarkers,
  type IChartApi,
  type ISeriesApi,
  type LineData,
  type AreaData,
  type Time,
  type SeriesMarker,
} from 'lightweight-charts';
import type { ChartSliderPosition, IndicatorConfig, PairHistory, PlotConfig, Trade } from '@/types';
import { ChartType } from '@/types';
import { computed, ref, watch, onMounted, onUnmounted } from 'vue';

const props = defineProps<{
  trades: Trade[];
  dataset: PairHistory;
  heikinAshi: boolean;
  showMarkArea: boolean;
  useUTC: boolean;
  plotConfig: PlotConfig;
  theme: 'dark' | 'light';
  sliderPosition?: ChartSliderPosition;
  colorUp: string;
  colorDown: string;
  labelSide: 'left' | 'right';
  startCandleCount: number;
}>();

const chartContainer = ref<HTMLDivElement>();
let chart: IChartApi | null = null;
let candlestickSeries: ISeriesApi<'Candlestick'> | null = null;
let volumeSeries: ISeriesApi<'Histogram'> | null = null;
const indicatorSeries: Map<string, ISeriesApi<'Line' | 'Area'>> = new Map();
const subplotSeries: Map<string, ISeriesApi<'Line' | 'Area'>> = new Map();
const signalMarkers: SeriesMarker<Time>[] = [];
const tradeMarkers: SeriesMarker<Time>[] = [];

const hasData = computed(() => {
  return props.dataset !== null && typeof props.dataset === 'object' && props.dataset.data.length > 0;
});

const pair = computed(() => (props.dataset ? props.dataset.pair : ''));
const timeframe = computed(() => (props.dataset ? props.dataset.timeframe : ''));

function getChartOptions() {
  const isDark = props.theme === 'dark';
  return {
    layout: {
      background: { type: 'solid', color: 'transparent' },
      textColor: isDark ? '#dedede' : '#333',
    },
    grid: {
      vertLines: { color: isDark ? '#1a1a1a' : '#f0f0f0' },
      horzLines: { color: isDark ? '#1a1a1a' : '#f0f0f0' },
    },
    crosshair: {
      mode: 1,
      vertLine: {
        color: '#cccccc',
        width: 1,
        style: 2,
        labelBackgroundColor: '#777',
      },
      horzLine: {
        color: '#cccccc',
        width: 1,
        style: 2,
        labelBackgroundColor: '#777',
      },
    },
    rightPriceScale: {
      borderColor: isDark ? '#2a2a2a' : '#e0e0e0',
      scaleMargins: {
        top: 0.1,
        bottom: 0.2,
      },
    },
    timeScale: {
      borderColor: isDark ? '#2a2a2a' : '#e0e0e0',
      timeVisible: true,
      secondsVisible: false,
    },
    handleScale: {
      axisPressedMouseMove: {
        time: true,
        price: true,
      },
    },
    handleScroll: {
      mouseWheel: true,
      pressedMouseMove: true,
      horzTouchDrag: true,
      vertTouchDrag: false,
    },
  };
}

function initializeChart() {
  if (!chartContainer.value || !hasData.value) return;

  if (chart) {
    chart.remove();
  }

  chart = createChart(chartContainer.value, getChartOptions());

  candlestickSeries = chart.addSeries(CandlestickSeries, {
    upColor: props.colorUp,
    downColor: props.colorDown,
    borderUpColor: props.colorUp,
    borderDownColor: props.colorDown,
    wickUpColor: props.colorUp,
    wickDownColor: props.colorDown,
  });

  volumeSeries = chart.addSeries(HistogramSeries, {
    priceFormat: { type: 'volume' },
    priceScaleId: '',
  });
  volumeSeries.moveToPane(1);
//   const secondPane = chart.panes()[1];
//   if (secondPane) {
//     secondPane.setHeight(100);
//   }

  volumeSeries.priceScale().applyOptions({
    scaleMargins: {
      top: 0.85,
      bottom: 0,
    },
  });

  updateChartData();

  if (props.startCandleCount > 0 && props.dataset.data.length > props.startCandleCount) {
    const visibleCount = Math.min(props.startCandleCount, props.dataset.data.length);
    chart.timeScale().setVisibleRange({
        from: new Date(props.dataset.data[props.dataset.data.length - visibleCount][0]).getTime() / 1000 as Time,
        to: new Date(props.dataset.data[props.dataset.data.length - 1][0]).getTime() / 1000 as Time,
    });
  }
}

function updateChartData() {
  if (!chart || !candlestickSeries || !volumeSeries || !hasData.value) return;

  const columns = props.dataset.columns.slice();
  const colDate = columns.findIndex((el) => el === '__date_ts');
  const colOpen = columns.findIndex((el) => el === 'open');
  const colHigh = columns.findIndex((el) => el === 'high');
  const colLow = columns.findIndex((el) => el === 'low');
  const colClose = columns.findIndex((el) => el === 'close');
  const colVolume = columns.findIndex((el) => el === 'volume');
  const colEnterTag = columns.findIndex((el) => el === 'enter_tag');
  const colExitTag = columns.findIndex((el) => el === 'exit_tag');
  const colEntryData = columns.findIndex((el) => el === '_buy_signal_close' || el === '_enter_long_signal_close');
  const colExitData = columns.findIndex((el) => el === '_sell_signal_close' || el === '_exit_long_signal_close');
  const colShortEntryData = columns.findIndex((el) => el === '_enter_short_signal_close');
  const colShortExitData = columns.findIndex((el) => el === '_exit_short_signal_close');

  let dataset = props.heikinAshi
    ? heikinAshiDataset(columns, props.dataset.data)
    : props.dataset.data.slice();

  dataset.forEach((row) => {


    const time = new Date(row[colDate]).getTime() / 1000 as Time; //(row[colDate] / 1000) as Time;

    const open = row[colOpen];
    const high = row[colHigh];
    const low = row[colLow];
    const close = row[colClose];
    const volume = row[colVolume];

    candlestickSeries?.update({ time, open, high, low, close });
    volumeSeries?.update({
      time,
      value: volume,
      color: close >= open ? props.colorUp : props.colorDown,
    });

    if (colEntryData >= 0 && row[colEntryData]) {
      signalMarkers.push({
        time,
        position: 'belowBar',
        color: '#00ff26',
        shape: 'arrowUp',
        text: row[colEnterTag] ? `Long Entry (${row[colEnterTag]})` : 'Long Entry',
      });
    }
    if (colExitData >= 0 && row[colExitData]) {
      signalMarkers.push({
        time,
        position: 'aboveBar',
        color: '#faba25',
        shape: 'arrowDown',
        text: row[colExitTag] ? `Long Exit (${row[colExitTag]})` : 'Long Exit',
      });
    }
    if (colShortEntryData >= 0 && row[colShortEntryData]) {
      signalMarkers.push({
        time,
        position: 'belowBar',
        color: '#00ff26',
        shape: 'arrowDown',
        text: 'Short Entry',
      });
    }
    if (colShortExitData >= 0 && row[colShortExitData]) {
      signalMarkers.push({
        time,
        position: 'aboveBar',
        color: '#faba25',
        shape: 'arrowUp',
        text: 'Short Exit',
      });
    }
  });

  createSeriesMarkers(candlestickSeries,signalMarkers);

  //updateTradeMarkers();
  //updateIndicators();
}

function updateTradeMarkers() {
  if (!hasData.value || !candlestickSeries) return;

  const filteredTrades = props.trades.filter((t) => t.pair === pair.value);
  const markers: SeriesMarker<Time>[] = [];

  filteredTrades.forEach((trade) => {
    console.log("trade.open_date_ts" + trade.open_date_ts)
    const entryTime = (trade.open_date_ts / 1000) as Time;
    const exitTime = trade.close_date_ts ? (trade.close_date_ts / 1000) as Time : undefined;

    if (trade.enter_tag?.toLowerCase().includes('short')) {
      markers.push({
        time: entryTime,
        position: 'belowBar',
        color: '#ff0000',
        shape: 'arrowDown',
        text: `Short ${trade.amount}`,
      });
    } else {
      markers.push({
        time: entryTime,
        position: 'belowBar',
        color: '#00ff26',
        shape: 'arrowUp',
        text: `Long ${trade.amount}`,
      });
    }

    if (exitTime) {
      markers.push({
        time: exitTime,
        position: 'aboveBar',
        color: '#faba25',
        shape: 'circle',
        text: `Close`,
      });
    }
  });

  createSeriesMarkers(candlestickSeries,markers);
}

function updateIndicators() {
  if (!chart || !hasData.value) return;

  indicatorSeries.forEach((series) => chart?.removeSeries(series));
  indicatorSeries.clear();
  subplotSeries.forEach((series) => chart?.removeSeries(series));
  subplotSeries.clear();

  const columns = props.dataset.columns.slice();
  const colDate = columns.findIndex((el) => el === '__date_ts');

  if ('main_plot' in props.plotConfig) {
    Object.entries(props.plotConfig.main_plot).forEach(([key, value]) => {
      const col = columns.findIndex((el) => el === key);
      if (col > 0) {
        const lineData: LineData<Time>[] = props.dataset.data.map((row) => ({
          time: new Date(row[colDate]).getTime() / 1000 as Time,//(row[colDate] / 1000) as Time,
          value: row[col],
        }));

        let series: ISeriesApi<'Line' | 'Area'>;
        if (value.fill_to) {
          const fillColKey = `${key}-${value.fill_to}`;
          const fillCol = columns.findIndex((el) => el === fillColKey);
          const areaData: AreaData<Time>[] = props.dataset.data.map((row) => ({
            time: new Date(row[colDate]).getTime() / 1000 as Time,//(row[colDate] / 1000) as Time,
            value: row[col],
          }));
          series = chart!.addSeries(AreaSeries, {
            lineColor: value.color,
            topColor: `${value.color}40`,
            bottomColor: 'transparent',
          });
          series.setData(areaData);
        } else {
          series = chart!.addSeries(LineSeries, {
            color: value.color,
            lineWidth: 2,
          });
          series.setData(lineData);
        }
        indicatorSeries.set(key, series);
      }
    });
  }

  if ('subplots' in props.plotConfig) {
    Object.entries(props.plotConfig.subplots).forEach(([key]) => {
      Object.entries(props.plotConfig.subplots[key]).forEach(([sk, sv]) => {
        const col = columns.findIndex((el) => el === sk);
        if (col > 0) {
          const lineData: LineData<Time>[] = props.dataset.data.map((row) => ({
            time: new Date(row[colDate]).getTime() / 1000 as Time,//(row[colDate] / 1000) as Time,
            value: row[col],
          }));
          const series = chart!.addSeries(LineSeries, {
            color: sv.color,
            lineWidth: 2,
          });
          series.setData(lineData);
          subplotSeries.set(`${key}-${sk}`, series);
        }
      });
    });
  }
}

/*
function updateSliderPosition() {
  if (!chart || !props.sliderPosition) return;

  const start = props.sliderPosition.startValue - props.dataset.timeframe_ms * 40;
  const end = props.sliderPosition.endValue
    ? props.sliderPosition.endValue + props.dataset.timeframe_ms * 40
    : props.sliderPosition.startValue + props.dataset.timeframe_ms * 80;

  chart.timeScale().setVisibleRange({
    from: (start / 1000) as Time,
    to: (end / 1000) as Time,
  });
}*/

let resizeObserver: ResizeObserver | null = null;

onMounted(() => {
  initializeChart();

  if (chartContainer.value) {
    resizeObserver = new ResizeObserver(() => {
      if (chart && chartContainer.value) {
        chart.applyOptions({
          width: chartContainer.value.clientWidth,
          height: chartContainer.value.clientHeight,
        });
      }
    });
    resizeObserver.observe(chartContainer.value);
  }
});

onUnmounted(() => {
  if (resizeObserver) {
    resizeObserver.disconnect();
  }
  if (chart) {
    chart.remove();
    chart = null;
  }
});

watch([() => props.dataset, () => props.heikinAshi, () => props.showMarkArea], () => {
  updateChartData();
});

watch([() => props.useUTC, () => props.theme, () => props.plotConfig], () => {
  if (chart) {
    chart.applyOptions(getChartOptions());
  }
  //updateIndicators();
});
/*
watch(
  () => props.sliderPosition,
  () => updateSliderPosition(),
);*/
</script>

<template>
  <div ref="chartContainer" class="h-full w-full"></div>
</template>

<style scoped>
div {
  min-height: 200px;
  height: 100%;
  width: 100%;
}
</style>