<template>
    <div class="main_graph" ref="graphContainer">
        <div id="info-window" class="float-info-window" v-if="info_window_str.length>0">
            <h3> {{ info_window_str }}</h3>
        </div>
        <div id="graphContainer" @resize="handleResize"></div>

        <div class="float-search-window">
            <input type="text" v-model.lazy="searchText" placeholder="Search" />
            <div>
                <div v-for="(value) in searchResult" @click="SelectNode(value, true)">
                    {{ value }}
                </div>
            </div>
        </div>
        <div class="float-node-info-window">
            <div v-if="selected_node_id" class="float-node-info-item">
                <strong>{{ selected_node_id }}</strong>
            </div>
            <div v-for="(value, key) in all_node_info[selected_node_id]" :key="key" class="float-node-info-item">
                <span v-if="['bound'].includes(key)">{{ key }}: {{ value }}</span>
                <span v-else-if="['inference_time'].includes(key)">{{ key }}: {{ strNumberTime(value) }}</span>
                <span v-else>{{ key }}: {{ strNumber(value) }}</span>
            </div>
            <div class="float-node-info-item">
                <canvas id="lineChart" width="300" height="200"></canvas>
            </div>
        </div>
    </div>
</template>

<script setup>
import G6 from "@antv/g6"

import { onMounted, onBeforeUpdate, provide } from 'vue'
import { watch, inject, ref } from 'vue'
import { graph_config } from "./graphs/graph_config.js"
// import { get_roofline_options } from "./graphs/roofline_config.js"
import axios from 'axios'
import { strNumber, strNumberTime } from '@/utils.js';
import { Chart, registerables } from 'chart.js';

import annotationPlugin from 'chartjs-plugin-annotation';

const model_id = inject('model_id')
const hardware = inject('hardware')
const global_update_trigger = inject('global_update_trigger')
const global_inference_config = inject('global_inference_config')
const ip_port = inject('ip_port')
const total_results = inject('total_results')
var hardware_info = {}


var graph = null;
var graph_data;
const all_node_info = ref({})
Chart.register(...registerables, annotationPlugin);

const searchText = ref('')
var searchResult = []

const selected_node_id = ref("")
var roofline_chart = null

const info_window_str = ref('')

const changeGraphSizeWaitTimer = ref(false);
window.onresize = () => {
    if (!changeGraphSizeWaitTimer.value & graph != null) {
        // console.log("handleResize", window.innerWidth, window.innerHeight)
        var leftControlDiv = document.querySelector('.left_control');
        var width = leftControlDiv.offsetWidth;
        graph.changeSize(window.innerWidth - width, window.innerHeight)
        changeGraphSizeWaitTimer.value = true;
        setTimeout(function () {
            changeGraphSizeWaitTimer.value = false;
        }, 100);
    }
};

function graphUpdate() {
    const url = 'http://' + ip_port.value + '/get_graph'
    console.log("graphUpdate", url)
    info_window_str.value="Loading from server..."
    var is_init=false
    axios.post(url, { model_id: model_id.value, hardware: hardware.value, inference_config: global_inference_config.value }).then(function (response) {
        console.log(response);
        info_window_str.value=""
        graph_data = response.data
        for (let i = 0; i < graph_data.nodes.length; i++) {
            all_node_info.value[graph_data.nodes[i].id] = graph_data.nodes[i].info;
        }
        total_results.value = response.data.total_results
        hardware_info = response.data.hardware_info

        const old_ids = new Set(graph.getNodes().map(node => node.get('id')));
        const new_ids = new Set(graph_data.nodes.map(node => node.id));
        const is_equal=old_ids.size === new_ids.size && [...old_ids].every(key => new_ids.has(key));

        if (is_equal) {
            // iterate each node
            graph_data.nodes.forEach(function (node) {
                // update the node
                graph.updateItem(node.id, {
                    description: node.description, label: node.label
                });
            });
        } else {
            graph.clear()
            graph.data(graph_data)
            graph.render()
        }
        console.log(graph_data)
        setTimeout(() => {
            update_roofline_model();
        }, 10);
        
        setTimeout(() => {
            graph.fitView();
        }, 10);

    })
        .catch(function (error) {
            info_window_str.value="Error in get_graph"
            console.log("error in graphUpdate");
            console.log(error);
        });

}

watch(() => global_update_trigger.value, () => graphUpdate(false))
// watch(() => global_update_trigger.value, () => update_roofline_model())
// watch(() => global_update_trigger.value, () => release_select())

function handleSearch(newText, oldText) {
    console.log("handleSearch", newText)
    const nodes = graph.findAll('node', (node) => {
        const nodeId = node.get('id');
        // console.log("handleSearch", node)
        return nodeId.includes(newText)
    });
    console.log("handleSearch", nodes)
    searchResult.length = 0
    for (let i = 0; i < nodes.length; i++) {
        const node = nodes[i];
        const nodeId = node.get('id');
        searchResult.push(nodeId)
        if (i > 100) {
            break
        }

    }
}
watch(searchText, handleSearch)

var nowFocusNode = null
var nowFocusNodePrevColor = null
function SelectNode(nodeId, moveView = false) {
    if (moveView) {
        graph.focusItem(nodeId, true)
    }
    if (nowFocusNode) {
        // console.log("nowFocusNodePrevColor", nowFocusNodePrevColor)
        nowFocusNode.update({
            style: {
                fill: nowFocusNodePrevColor,
            },
        });
    }
    const node = graph.findById(nodeId)
    if (node) {
        // 高亮
        if (node.getModel().style.fill) {
            nowFocusNodePrevColor = node.getModel().style.fill
        } else {
            nowFocusNodePrevColor = "#ffffff"
        }
        node.update({
            style: {
                fill: "#dffdff",
            },
        });
        nowFocusNode = node
    }

    selected_node_id.value = nodeId
}


function update_roofline_model() {
    const ctx = document.getElementById('lineChart');
    if (ctx) {
        if (roofline_chart) {
            roofline_chart.destroy();
        }
        const bandwidth = hardware_info["bandwidth"];
        const max_OPS = hardware_info["max_OPS"];
        const turningPoint = max_OPS / bandwidth;

        var annotation
        var x_max
        if (selected_node_id.value){
            const node_arithmetic_intensity = all_node_info.value[selected_node_id.value]["arithmetic_intensity"];
            x_max = Math.max(turningPoint * 3, node_arithmetic_intensity+1);
            annotation={
                        annotations: {
                            lineX: {
                                type: 'line',
                                xMin: node_arithmetic_intensity,
                                xMax: node_arithmetic_intensity,
                                yMin: 0,
                                yMax: max_OPS * 1.1,
                                borderColor: 'blue',
                                borderWidth: 2,
                                borderDash: [5, 5], // 虚线样式
                                label: {
                                    enabled: true,
                                    content: 'Node AI',
                                    position: 'top'
                                }
                            }
                        }
                    }
        }else{
            annotation={}
            x_max = turningPoint * 3
        }
        roofline_chart = new Chart(ctx, {
            type: 'line',
            data:
            {
                // labels: [0, turningPoint, 321],
                datasets: [{
                    label: 'Roofline',
                    data: [
                        { x: 0, y: 0 },
                        { x: turningPoint, y: max_OPS },
                        { x: x_max, y: max_OPS }
                    ],
                    // [0, max_OPS, max_OPS],
                    borderColor: 'black',
                    borderWidth: 2,
                    fill: false,
                    pointRadius: 0 // 不显示数据点
                }]
            },
            options: {
                responsive: true,
                maintainAspectRatio: false,

                scales: {
                    x: {
                        title: {
                            display: true,
                            text: 'Arithmetic Intensity (OPs/byte)'
                        },
                        type: 'linear',
                        ticks: {
                            callback: function (value, index, values) {
                                return value.toFixed(1);
                            }
                        },
                        beginAtZero: true,
                        max: x_max
                    },
                    y: {
                        title: {
                            display: true,
                            text: 'Performance (OPS)'
                        },
                        ticks: {
                            callback: function (value, index, values) {
                                return value.toExponential(1);
                            }
                        },
                        beginAtZero: true,
                        max: max_OPS * 1.1
                    }
                },
                plugins: {
                    title: {
                        display: true,
                        text: 'Roofline Model', // 这里是你想要的标题
                        position: 'top' // 标题的位置，可以是'top', 'left', 'bottom', 或 'right'
                    },
                    legend: {
                        display: false
                    },
                    annotation: annotation
                }
            }
        });
    }
}

function release_select(){
    selected_node_id.value = ""
    update_roofline_model()
}

onMounted(() => {
    graph = new G6.Graph(graph_config); 
    graph.on('node:click', (event) => {
        const { item } = event;
        const node = item.getModel();
        clickNode(node);
    });
    graph.on('node:touchstart', (event) => {
        const { item } = event;
        const node = item.getModel();
        clickNode(node);
    });
    graph.on('canvas:click', (event) => {
        release_select()
    });
    graphUpdate(true);
    graph.render();
    
    
})

function clickNode(node) {
    console.log(node);
    const nodeId = node.id;
    SelectNode(nodeId);
    // sleep 100ms
    setTimeout(() => {
        update_roofline_model();
    }, 100);
}
</script>

<style scoped>
.main_graph {
    width: 75%;
    height: 100%;

    position: relative;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    float: right;
    /* width: 85%; */
    flex-grow: 1;
    background-color: #ffffff;
    border: 0px;
}

.float-search-window {
    position: absolute;
    top: 10px;
    right: 10px;
    height: auto;
    max-height: 50vh;
    background-color: #f1f1f1b7;
    padding: 3px;
    overflow-y: auto;
}


.float-info-window {
    position: absolute;
    top: 10px;
    left: 40%;
    height: auto;
    width: 20%;
    background-color: #f1f1f1b7;
    padding: 5px;
    overflow-y: auto;
}

.float-node-info-window {
    position: absolute;
    top: 10px;
    left: 10px;
    background-color: #f1f1f1b7;
}

.float-node-info-item {
    padding: 3px;
    border-top: 1px solid #e2e2e2;
}
</style>