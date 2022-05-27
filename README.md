<!-- 
Skyhawk Flight Instruments (https://github.com/uw-ray/Skyhawk-Flight-Instruments)
By Raymond Blaga (raymond.blaga@gmail.com), Edward Hanna (edward.hanna@senecacollege.ca), Pavlo Kuzhel (pavlo.kuzhel@senecacollege.ca)

Forked from jQuery Flight Indicators (https://github.com/sebmatton/jQuery-Flight-Indicators)
By Sébastien Matton (seb_matton@hotmail.com)

Published under GPLv3 License.
-->


<meta charset="utf-8">
<html lang="en" dir="ltr">

<head>
    <title>Attitude Indicator</title>
    <script src="js/jquery-1.11.3.js"></script>
    <script src="js/d3.min.js"></script>
    <script src="js/jquery.flightindicators.js"></script>
    <link rel="stylesheet" type="text/css" href="css/flightindicators.css" />
    <script src="https://maxcdn.bootstrapcdn.com/bootstrap/3.4.0/js/bootstrap.min.js"></script>
    <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.4.0/css/bootstrap.min.css">
    <script src="js/echarts.js"></script>
    <style>
      .n {
		margin: auto;
        width: 400px;
        height: 10px;
		border-radius: 10px;
        background-color: #748ee4;
        position: relative;
      }
      progress {
		margin: auto;
        background: #ccc;
        width: 400px;
        height: 10px;
        left: 0px;
		top : -2px;
        border: 1px solid #ddd;
        border-radius: 10px;
        cursor: pointer;
        position: relative;
      }
      /* 表示已完成进度背景色,仅兼容谷歌浏览器 */
      progress::-webkit-progress-value {
        background: #748ee4;
        height: 10px;
        border-radius: 10px 0 0 10px;
        position: absolute;
        top: -1px;
      }
      /* 表示总任务的背景色 ，仅兼容谷歌浏览器 */
      progress::-webkit-progress-bar {
        background: #ccc;
      }
      .progress_btn {
        margin-left: 0px;
        width: 20px;
        height: 20px;
        background: #ffffff;
        border: 1px solid #ccc;
        box-shadow: 0 0 0 0 #000 ;
        border-radius: 50%;
        position: absolute;
        top: -5px;
        cursor: pointer;
      }
    </style>
</head>

<body>
    <center>
    <div class="shang">
    <h1>模拟飞机姿态仪</h1>
    </div>
    <div>
        <span id="attitude"></span>
    </div>
    <div>
        <br>
        <form id="fileSelectionForm" style="left: 20px; width: 200px; white-space: nowrap;">
            <input type="button" id="loadFile" value="选择飞行数据文件" onclick="document.getElementById('dataFile').click();" />
            <input type="file" style="display:none;" id="dataFile" accept=".csv" name="dataFile" onchange="inputData(this)"/>
            <span id="dataFileName">  </span>
        </form>
        <br>
    </div>
    <div>
        <div class="container" style="left: 0px; width: 600px;">
            <button type="button" id="playBtn" class="btn btn-default">
            <span id="playIcon" class="glyphicon glyphicon-play"></span></button>
            <span id="speedSelection" style="float: right">
                <span>播放速度</span>
                <label class="radio-inline">
                    <input type="radio" value=1 name="playSpeed" checked>正常
                </label>
                <label class="radio-inline">
                    <input type="radio" value=5 name="playSpeed">X5
                </label>
                <label class="radio-inline">
                    <input type="radio" value=10 name="playSpeed">X10
                </label>
                <label class="radio-inline">
                    <input type="radio" value=25 name="playSpeed">X25
                </label>
                <label class="radio-inline">
                    <input type="radio" value=50 name="playSpeed">X50
                </label>
`            </span>
        </div>
	<br>
        <div id="dataChart" style="width: 500px;height:200px;">
        </div>
        <div class="n">
			<progress id="progress" value="0"> </progress>
			<div id="probnt" class="progress_btn"></div>
		</div>
    </div>
    </center>

<!--    <div class="progress progress-striped active" style="width: 400px;">-->
<!--    <div class="progress-bar progress-bar-info" role="progressbar"-->
<!--         aria-valuenow="60" aria-valuemin="0" aria-valuemax="100"-->
<!--         style="width: 40%;">-->
<!--    </div>-->


    <script id="DataFileProcession">
        // 全局变量声明
        // 数据参数
        var keyTime = "";
        var keyPitch = "";
        var keyRoll = "";
        
        //进度条参数
        var pro;
        var flag = true; //状态值，按下时才能拖拽
        var probnt = document.getElementById("probnt");
        var progress = document.getElementById("progress");
        var body = document.body;
        var offsetX;
        var X;
        // 数据文件参数
        var fileName = ""; //数据文件名
        var flightData = ""; //转成数组的飞行数据

        // chart 参数
        var option = "";
        var dataPitch;
        var dataRoll;
        var dataTime;
        var myChart = echarts.init(document.getElementById('dataChart'));
        var dataLen = 0; //数据长度
        var currentIdx = 0; //当前显示的数据位置
        var displayTip = true; //显示播放坐标处的tip
        var tempParam = 0;

        // 播放控制
        var dataReady = false; //数据是否已经准备好
        var playStatus = false; //开始处于不播放的状态
        var speedUp = 1; //播放速度
        const dataFPS = 50; //刷新率50Hz
        const timeGap = 20; //显示时间间隔
        var sampleCount = 0; //采样序号
        var diffPitch = 0; //下一秒数据和当前秒数据的差异
        var diffRoll = 0;
        var currentPitch; //当前时间坐标的IAS
        var currentRoll;
        var stepPitch; //IAS 递增步长
        var stepRoll;


        // 仪表显示
        var attitude = $.flightIndicator('#attitude', 'attitude', {roll:0, pitch:0});
        attitude.setOffFlag(false)
		attitude.setILS(false)

		// 将csv文件当作string读取后，转换成array (功能正确 200-5-11)
		function csvToArray(str, delimiter = ",") {
            // 飞行数据文件第三行是数据名称，第四行开始是数据
            const lineOneEnd = str.indexOf("\n");
            const lineTwoEnd = str.indexOf("\n", lineOneEnd+1);
            const lineThreeEnd = str.indexOf("\n", lineTwoEnd+1);
            const headers = str.slice(lineTwoEnd+1, lineThreeEnd).split(delimiter);
    		// 所有需要使用到的数据的键值
            // 获取需要的键值，根据对应列的序号
            keyTime = headers[1];
	    keyPitch = headers[13];
	    keyRoll = headers[14];
            
            //将str转换成数组
            const rows = str.slice(lineThreeEnd + 1).split("\n");
            rows.pop(); //数据最后一行不要
	    dataLen = rows.length;
            const arr = rows.map(function (row) {
            const values = row.split(delimiter);
            const el = headers.reduce(function (object, header, index) {
            object[header] = values[index];
            return object;
            }, {});
                return el;
            });
            // return the array
            return arr;
		}

        // 指定图表的配置项和数据
        function setEchartOption(axisX,axisY1,axisY2){
            option = {
                title: {
                  text: 'Flight Attitude Data'
                },
                tooltip: {
                    trigger: 'axis',
                    axisPointer: {
                        animation: false
                    },
	            backgroundColor: 'rgba(255,255,255,0)',// 提示框浮层的背景颜色。Alpha参数是一个介于0.0（完全透明）和1.0（完全不透明）之间的参数。
                    borderColor: '#000000', // 提示框浮层的边框颜色。
                    textStyle: { // 提示框浮层的文本样式。
                    color: '#000000',
                    },
                    formatter: (params) => {
                        let result = params[0].axisValueLabel + '<br>'
                        params.forEach(function (item) {
                            if (item.value) {
                            result += item.marker + ' ' + item.seriesName + ' : ' + item.value + '</br>'
                            }
                        })
                    // 保留数据
                    tempParam = params[0]
                    // 返回mousemove提示信息
                    return result
                    }
                },
                legend: {
                  data: ['Pitch', 'Roll']
                },
                toolbox: {
			    show : true,
                feature: {
				dataView: {show: true, readOnly: false},
				magicType: {show: true, type: ['line', 'bar']},
				restore: {},
				saveAsImage: {},
			     }
                     },
                xAxis: {
                  data: axisX
                },
                yAxis: {
                    type: 'value'
                },
                series: [
                  {
                    name: 'Pitch',
                    type: 'line',
                    data: axisY1
                  },
                  {
                    name: 'Roll',
                    type: 'line',
                    data: axisY2
                  }
                ],
            };
        }

        // 数据波形图显示
        var reader = new FileReader();
        reader.onload = function () {
            console.log("reader.onload");
            flightData = reader.result;
            // console.log(flightData.slice(0,10));
            let data = csvToArray(flightData); //TBD:处理数据文件最后一行可能不完整的情况，目前先保证文件组后一行完整
            // 提取要显示的数据
            dataPitch = data.map(function (item){return Number(item[keyPitch])});
            dataRoll = data.map(function (item){return Number(item[keyRoll])});
            dataTime = data.map(function (item){return item[keyTime]});
            //console.log(dataTime[1]);
            // 数据转换完之后，显示数据的波形图，使能仪表播放控制控件
            setEchartOption(dataTime, dataPitch, dataRoll);
            myChart.setOption(option);
            // 在波形图中点击鼠标，将数据播放的位置设定到当前位置
            myChart.getZr().on('click', params => {
 	            const pointInPixel = [params.offsetX, params.offsetY]
                if (myChart.containPixel('grid', pointInPixel)) {
                    // console.log("拿到当前点击节点的索引",tempParam.dataIndex)
                    currentIdx = tempParam.dataIndex;
                    sampleCount = 0;
                }
            });
        };

        //播放按键控制
        document.getElementById("playBtn").disabled = true;
        var playButton = document.getElementById("playBtn");
        var playSpan = document.getElementById("playIcon");
        playSpan.innerHTML = "Play";
        playButton.addEventListener("click", function(e) {
            //切换状态，并改变图标
            if (playStatus == false) {
                playStatus = true;
                playSpan.className = "glyphicon glyphicon-pause";
                playSpan.innerHTML = "Pause"
                sampleCount = 0; //总是从整秒数据开始播放
            }
            else{
                console.log("pause button click")
                playStatus = false;
                let spans = playButton.getElementsByTagName("span");
                playSpan.className = "glyphicon glyphicon-play";
                playSpan.innerHTML = "Play"
                sampleCount = 0; //总是从整秒数据开始播放
            }
        })

        // 播放速度选择响应
        document.getElementById("speedSelection").addEventListener("click", function(e) {
            if (e.target.tagName === "INPUT") {
                // console.log("radio value", e.target.value)
                speedUp = Number(e.target.value);
                sampleCount = 0; //重新回到当前
            }
        })

        // 数据播放
        setInterval(function() {

            //如果当前处于播放状态，则根据更新率显示
            if(playStatus == true){
                // 当遇到整秒数据时，计算下一秒数据到当前秒的差值，并显示当前时间点的波形显示的tip
                if(sampleCount == 0){
                    currentPitch = 0.8 * dataPitch[currentIdx];
                    diffPitch = 0.8 * dataPitch[currentIdx+1] - currentPitch;
                    stepPitch = diffPitch/50;
                    displayTip = true;
                    currentRoll = -dataRoll[currentIdx];
                    diffRoll = -dataRoll[currentIdx+1] - currentRoll;
                    stepRoll = diffRoll/50;
                    displayTip = true;
                }
                //如果不是整数秒，计算插值，更新仪表显示
                else{
                    currentPitch = 0.8 * dataPitch[currentIdx] + stepPitch*sampleCount;
                    currentRoll = -dataRoll[currentIdx] + stepRoll*sampleCount;
                }
                attitude.setPitch(currentPitch);
                attitude.setRoll(currentRoll);
                //console.log("idx=",currentIdx," step=", stepIAS, " count=", sampleCount, " ias=", currentIAS);
                //向后递进一步
                sampleCount = (sampleCount+speedUp)%50;
                if(sampleCount == 0){currentIdx++;}
                //到数据最后，将数据定位到最开始，并停止动态显示
                if(currentIdx==dataLen){
                    currentIdx = 0;
                    playStatus = false;
                }
            }
            else {
                displayTip = false;
            }
            if(displayTip == true){
                myChart.dispatchAction({
                        type: 'showTip',
                        seriesIndex: 0,
                        dataIndex: currentIdx
                });
            }
                progress.max = dataLen;
		        progress.value = currentIdx;
                probnt.style.marginLeft = (currentIdx/dataLen)*380 + "px";

                probnt.onmousedown = function (e) {
					flag = true;
					offsetX = e.offsetX;   //获取鼠标按下时的位置坐标
                     };

				body.onmousemove = function (e) {
                if (flag) {
					X = e.clientX;  //获取当前鼠标的位置
                    let cl = X - offsetX;
                    const { left, width } = progress.getBoundingClientRect();
                    var ml = cl - left;
                    if (ml <= 0) {
                     probnt.style.marginLeft = 0 + "px";
                   } else if (ml >= width - 20) {
                     probnt.style.marginLeft = width - 20 + "px";
                    } else {
                     probnt.style.marginLeft = ml + "px"; //这里的20是progress的left值
                   }
                     progress.value = (ml/380)*dataLen;
                     currentIdx = Math.round((ml/380)*dataLen);
                        }
                      };
                     body.onmouseup = function () {
                       flag = false;
                         };

        }, timeGap);

        <!-- 打开数据文件后，将数据载入-->
		function inputData(selectedFile) {
           // console.log("inputData")
            let csvFile = selectedFile.files[0];
            fileName = csvFile.name;
            document.getElementById("dataFileName").innerHTML = fileName;
            reader.readAsText(csvFile);
            //数据准备好后，播放是能
            dataReady = true;
            document.getElementById("playBtn").disabled = false;
            console.log("enable button")
        }


  </script>

</body>

</html>
