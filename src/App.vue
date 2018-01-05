<template>
<div>
  <div class="row">
    <div class="col-md-1"></div>
    <div class="col-md-10">
      <div id="container" style="height: 300px; width: 100%;">
      </div>
    </div>
    <div class="col-md-1"></div>
  </div>
  <div class="row">
    <div class="col-md-1"></div>
    <div class="col-md-10" v-show=filteredInvoice.length>
      <table class="table table-striped table-bordered">
      <thead>
        <tr>
          <th class="col-md-1">Reference Number</th>
          <th class="col-md-1">Date</th>
          <th class="col-md-1">Face Value in USD</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="inv in filteredInvoice" :key="inv.maturityDate">
          <td class="col-md-1">{{inv.referenceNumber}}</td>
          <td class="col-md-1">{{inv.maturityDate}}</td>
          <td class="col-md-1">{{inv.faceValueInUSD}}</td>
        </tr>
      </tbody>
      </table>
    </div>
    <div class="col-md-1"></div>
  </div>
</div>
</template>

<script lang="ts">
import Vue from "vue";
import Component from "vue-class-component";
import * as echarts from "echarts";
import { IInvoiceValue } from "./interface/IInvoiceValue";
import Axios from "axios";

@Component
export default class App extends Vue {
  filteredInvoice: Array<IInvoiceValue> = [];
  invoiceList: Array<IInvoiceValue> = [];

  created() {
    document.addEventListener("dateTapped", this.handleDateTapped);
  }

  destroyed() {
    document.removeEventListener("dateTapped", this.handleDateTapped);
  }

  handleDateTapped(event) {
    console.log("Date Tapped: " + event.detail.dateTapped);
    this.filteredInvoice = this.invoiceList.filter(function(row) {
      if (row.maturityDate === event.detail.dateTapped) {
        return true;
      } else {
        return false;
      }
    });
    console.log("Number of selected invoices: " + this.filteredInvoice.length);
  }

  mounted() {
    var dates: Array<String> = [];
    var values: Array<number> = [];
    var numbers: Array<number> = [];

    const ec = echarts as any;
    const container = document.getElementById("container");
    console.log("echarts container: " + container);

    const chart = ec.init(container);

    const option = {
      color: ["#3398DB", "#db0406"],
      tooltip: {
        trigger: "axis",
        axisPointer: {
          type: "shadow" // Option：'line' | 'shadow'
        }
      },
      legend: {
        data: ["Face Value in USD", "Number of Invoices"]
      },
      grid: {
        left: "3%",
        right: "4%",
        bottom: "3%",
        containLabel: true
      },
      xAxis: {
        type: "category",
        data: dates
      },
      yAxis: [
        {
          type: "value",
          name: "Face Value in USD"
        },
        {
          type: "category",
          name: "Number of Invoices",
          show: false
        }
      ],
      series: [
        {
          name: "Face Value in USD",
          type: "bar",
          stack: "total",
          label: {
            normal: {
              show: true,
              position: "insideBottom"
            }
          },
          data: values
        },
        {
          name: "Number of Invoices",
          type: "bar",
          stack: "total",
          scale: true,
          label: {
            normal: {
              show: true,
              position: "outsideTop"
            }
          },
          data: numbers
        }
      ]
    };

    chart.on("click", function(params: any) {
      console.log(params);
      console.log("Date - " + params.name + " be tapped.");

      var old_event: Event = params.event.event;
      old_event.stopPropagation();

      var event = new CustomEvent("dateTapped", {
        detail: { dateTapped: params.name }
      });
      document.dispatchEvent(event);
    });

    Axios.get(`http://10.39.101.14:9080/rest/invoice`)
      .then(response => {
        // JSON responses are automatically parsed.
        this.invoiceList = response.data;
        console.log("Invoices number: " + this.invoiceList.length);

        //face values and numbers be grouped by date
        var result = [];
        this.invoiceList.reduce(function(res, value) {
          if (!res[value.maturityDate]) {
            res[value.maturityDate] = {
              qty: 0,
              x: value.maturityDate,
              y: 0
            };
            result.push(res[value.maturityDate]);
          }
          res[value.maturityDate].qty += 1;
          res[value.maturityDate].y += value.faceValueInUSD;
          return res;
        }, {});

        //construct data points
        result.forEach(function(item) {
          dates.push(item.x);
          values.push(item.y);
          numbers.push(item.qty);
        });

        console.log(
          "There are " +
            dates.length +
            " dates and " +
            this.invoiceList.length +
            " invoices to be shown."
        );

        chart.setOption(option);
      })
      .catch(e => {
        console.log(e);
      });
  }
}
</script>
