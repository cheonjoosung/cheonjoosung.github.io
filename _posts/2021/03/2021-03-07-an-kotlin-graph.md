---
title: (안드로이드/코틀린) 바차트 그래프 그리기 - Bar Chart Graph
tags: [Android, Kotlin]
style: fill
color: dark
description: 안드로이드 코틀린에서 바차트 그래프 그리기 - Bar Chart Graph
---

## 안드로이드 Bar Chart 그리기
안드로이드에서 데이터 및 정보를 그래프로 표시해야 하는 경우가 종종 있는데 안드로이드 스튜디오 내에서 제공하는 기능만으로 표현하기가 쉽지 않다. 그래서 깃헙 오픈소스인 [MPAndroidChart](https://github.com/PhilJay/MPAndroidChart)를 활용하는 것을 추천한다. 여러 그래프가 존재하지만 그 중 흔한 Bar Chart 샘플을 구현해봅시다. 해당 깃헙에 나오는 docs 는 따라하기 까다로우므로 무작정 따라하지 말고 각 변수명이 뜻하는 바를 이해한다음 따라하는 것이 정신적인 건강에 좋음
![preview](https://camo.githubusercontent.com/01325d058289958fb40415facdc02d5209de23a1a03dab6815bd1d06154767e5/68747470733a2f2f7261772e6769746875622e636f6d2f5068696c4a61792f4d5043686172742f6d61737465722f73637265656e73686f74732f73696d706c6564657369676e5f6261726368617274332e706e67)


## 실습하기
**1. Gradle 종속성 추가**
- 모바일 뿐만 아니라 다른 파트의 많은 개발자들이 Utils 클래스를 만들고 그 안에 자주 쓰이는 코드들을 넣고 다니며 프로젝트 개발 속도를 늘린다.

```javascript
//app gradle
dependencies {
    //생략
    // - - Statistics for walk
    implementation 'com.github.PhilJay:MPAndroidChart:v3.1.0'
}

//project
repositories {
    maven { url 'https://jitpack.io' }
}
```

- Gradle APP 과 Gradle Project 에 코드를 추가

**2. 레이아웃 추가**

```javascript

<com.github.mikephil.charting.charts.BarChart
            android:id="@+id/chart_week"
            android:layout_width="match_parent"
            android:layout_height="300dp"
            android:layout_marginStart="16dp"
            android:layout_marginTop="8dp"
            android:layout_marginEnd="16dp" />

```
- 폭, 높이, 마진을 선호에 따라 변경


**3. Activity 코드 추가**
```javascript
private fun setChartView(view: View) {
    var chartWeek = view.findViewById<BarChart>(R.id.chart_week)
    setWeek(chartWeek)
}

private fun initBarDataSet(barDataSet: BarDataSet) {
    //Changing the color of the bar
    barDataSet.color = Color.parseColor("#304567")
    //Setting the size of the form in the legend
    barDataSet.formSize = 15f
    //showing the value of the bar, default true if not set
    barDataSet.setDrawValues(false)
    //setting the text size of the value of the bar
    barDataSet.valueTextSize = 12f
}

private fun setWeek(barChart: BarChart) {
    initBarChart(barChart)

    barChart.setScaleEnabled(false) //Zoom In/Out

    val valueList = ArrayList<Double>()
    val entries: ArrayList<BarEntry> = ArrayList()
    val title = "걸음 수"

    //input data
    for (i in 0..5) {
        valueList.add(i * 100.1)
    }

    //fit the data into a bar
    for (i in 0 until valueList.size) {
        val barEntry = BarEntry(i.toFloat(), valueList[i].toFloat())
        entries.add(barEntry)
    }
    val barDataSet = BarDataSet(entries, title)
    val data = BarData(barDataSet)
    barChart.data = data
    barChart.invalidate()
}

private fun initBarChart(barChart: BarChart) {
    //hiding the grey background of the chart, default false if not set
    barChart.setDrawGridBackground(false)
    //remove the bar shadow, default false if not set
    barChart.setDrawBarShadow(false)
    //remove border of the chart, default false if not set
    barChart.setDrawBorders(false)

    //remove the description label text located at the lower right corner
    val description = Description()
    description.setEnabled(false)
    barChart.setDescription(description)

    //X, Y 바의 애니메이션 효과
    barChart.animateY(1000)
    barChart.animateX(1000)


    //바텀 좌표 값
    val xAxis: XAxis = barChart.getXAxis()
    //change the position of x-axis to the bottom
    xAxis.position = XAxis.XAxisPosition.BOTTOM
    //set the horizontal distance of the grid line
    xAxis.granularity = 1f
    xAxis.textColor = Color.RED
    //hiding the x-axis line, default true if not set
    xAxis.setDrawAxisLine(false)
    //hiding the vertical grid lines, default true if not set
    xAxis.setDrawGridLines(false)


    //좌측 값 hiding the left y-axis line, default true if not set
    val leftAxis: YAxis = barChart.getAxisLeft()
    leftAxis.setDrawAxisLine(false)
    leftAxis.textColor = Color.RED


    //우측 값 hiding the right y-axis line, default true if not set
    val rightAxis: YAxis = barChart.getAxisRight()
    rightAxis.setDrawAxisLine(false)
    rightAxis.textColor = Color.RED


    //바차트의 타이틀
    val legend: Legend = barChart.getLegend()
    //setting the shape of the legend form to line, default square shape
    legend.form = Legend.LegendForm.LINE
    //setting the text size of the legend
    legend.textSize = 11f
    legend.textColor = Color.YELLOW
    //setting the alignment of legend toward the chart
    legend.verticalAlignment = Legend.LegendVerticalAlignment.TOP
    legend.horizontalAlignment = Legend.LegendHorizontalAlignment.CENTER
    //setting the stacking direction of legend
    legend.orientation = Legend.LegendOrientation.HORIZONTAL
    //setting the location of legend outside the chart, default false if not set
    legend.setDrawInside(false)
}
```
- [외국분의 코드 참조](https://medium.com/@clyeung0714/using-mpandroidchart-for-android-application-barchart-540a55b4b9ef)
- 가장 먼저 바차트의 색상/크기/텍스트 등 기본 설정 후 -> 바차트의 데이터 값 설정 -> 바차트 좌/우/아래에 표시되는 값 설정 -> 바차트의 타이틀 설정

## 결론
- **복 붙**
