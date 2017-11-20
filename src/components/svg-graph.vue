<template id="svg-graph">
    <div>
        <div class="pc-graph-controls">
            <label class="pc-graph-radio-label">
                <input type="radio" class="pc-graph-radio-input" name="graph-x" value="sample-power" v-model="graphType">
                <span class="pc-graph-radio-text" :class="{'pc-graph-radio-selected': graphType == 'sample-power'}">{{getMetricDisplayName('sample')}}</span>
            </label>
            <label class="pc-graph-radio-label">
                <input type="radio" class="pc-graph-radio-input" name="graph-x" value="impact-power" v-model="graphType">
                <span class="pc-graph-radio-text" :class="{'pc-graph-radio-selected': graphType == 'impact-power'}">{{getMetricDisplayName('impact')}}</span>
            </label>
            <label class="pc-graph-radio-label">
                <input type="radio" class="pc-graph-radio-input" name="graph-x" value="sample-impact" v-model="graphType">
                <span class="pc-graph-radio-text" :class="{'pc-graph-radio-selected': graphType == 'sample-impact'}">{{getMetricDisplayName('impact')}} vs {{getMetricDisplayName('sample')}}</span>
            </label>
        </div>
        <div>
            <div v-bind:style="style" ref="pc-graph-wrapper">
                <div ref="pc-graph"></div>
            </div>
        </div>
    </div>
</template>


<script>

import statFormulas from '../js/math.js'
import valueTransformationMixin from '../js/value-transformation-mixin.js'

let dataDefault = [
        ['x', 0, 0, 0, 0, 0, 0],
        ['Sample', 0, 0, 0, 0, 0],
        ['Current', null, null, 50]
    ];

let style = document.createElement('style');

style.innerHTML = `
    .c3-circles-Sample {
        display: none;
    }

    .c3-axis-y-label {
        pointer-events: none;
    }
`;

document.querySelector('head').appendChild(style);



export default {
    mixins: [valueTransformationMixin],
    template: '#svg-graph',
    props: ['testtype', 'sample', 'impact', 'power', 'base', 'falseposrate', 'sdrate'],
    data () {
        return {
            width: 100,
            height: 100,
            data:  this.dataDefault,
            graphType: 'sample-power' // x, y
            // graphX: 'sample' // computed
            // graphY: 'power' // computed
        }
    },
    computed: {
        style () {
            let { width, height } = this;

            return {
                width: `${width}px`,
                height: `${height}px`
            }
        },
        math () {
            return statFormulas[this.testtype]
        },
        graphX () {
            // 'sample'
            return this.graphType.split('-')[0]
        },
        graphY () {
            // 'power'
            return this.graphType.split('-')[1]
        },
    },
    methods: {
        resize () {
            let {width, height} = window.getComputedStyle(this.$el.parentNode);

            // update svg size
            this.width = window.parseInt(width);
            this.height = window.parseInt(height);
        },
        createYList ({ amount, rate = 10,  cur }) { //rate of 10 and amount of 10 will reach from 0 to 100
            let result = [];

            for (let i = 0; i <= amount; i++) {
                let y = rate * i,
                    nextY = rate * (i + 1);

                result.push(y);

                if (cur > y && cur < nextY) {
                    result.push(cur);
                }
            }

            return result;
        },
        trimInvalidSamples (newData) {
            let result = newData[0].reduce((prevArr, xValue, i) => {

                if (i == 0) {
                    prevArr[0] = [];
                    prevArr[1] = [];
                    prevArr[2] = [];
                }

                // i == 0 is the name of the dataset
                if (i == 0 || (!isNaN(xValue) && isFinite(xValue))) {
                    prevArr[0].push(newData[0][i]);
                    prevArr[1].push(newData[1][i]);
                    prevArr[2].push(newData[2][i]);
                }

                return prevArr;
            }, [])
            return result;
        },
        updateGraphData () {

            let clonedValues = this.deepCloneObject(this.convertDisplayedValues()),
                newData = this.deepCloneObject(dataDefault),
                curPower = window.parseInt(this.power),
                curImpact = window.parseInt(this.impact),
                yList,
                curY;



            if (this.graphY == 'power') {
                yList = this.createYList({ amount: 10, cur: curPower });
                curY = curPower;
            } else if (this.graphY == 'impact') {
                yList = this.createYList({ amount: 10, rate: 2, cur: curImpact });
                curY = curImpact;
            }

            // erase previous values but keep names of datasets
            newData[0].length = 1;
            newData[1].length = 1;
            newData[2].length = 1;

            yList.forEach((yValue, i) => {

                if (this.graphY == 'power') {
                    clonedValues.beta = 1 - this.extractValue(this.graphY, yValue);
                } else if (this.graphY == 'impact') {
                    clonedValues.effect_size = this.extractValue(this.graphY, yValue);
                }


                let xValues = this.displayValue(this.graphX, (this.math[this.graphX](clonedValues)));

                 newData[0][i + 1] = xValues; // x
                 newData[1][i + 1] = yValue; // line
                 newData[2][i + 1] = yValue == curY ? yValue : null; // current power dot

            })

            newData = this.trimInvalidSamples(newData);

            this.chart.axis.labels({x: this.updateXLabel(), y: this.updateYLabel()});

            this.chart.load({
                columns: newData
            })
        },
        convertDisplayedValues () {
            let { extractValue } = this,
                { sample, base, impact, falseposrate, power, sdrate } = this;

            return {
                total_sample_size: extractValue('sample', sample),
                base_rate: extractValue('base', base),
                effect_size: extractValue('impact', impact),
                alpha: extractValue('falsePosRate', falseposrate),
                beta: 1 - extractValue('power', power), // power of 80%, beta is actually 20%
                sd_rate: extractValue('falsePosRate', sdrate)
            }
        },
        deepCloneObject (obj) {
            return JSON.parse(JSON.stringify(obj))
        },
        createTooltip (V) {

            return {
                grouped: false,
                contents ([{x = 0, value = 0, id = ''}]) {

                    let {graphX, graphY, getMetricDisplayName} = V,
                        th = getMetricDisplayName(graphX),
                        name = getMetricDisplayName(graphY);

                    return `
                        <table class="c3-tooltip">
                            <tbody>
                                <tr>
                                    <th colspan="2">${th}: ${x}</th>
                                </tr>
                                <tr class="c3-tooltip-name--Current">
                                    <td class="name">
                                        <span style="background-color:#ff7f0e">
                                        </span>
                                        ${name}
                                    </td>
                                    <td class="value">${value}</td>
                                </tr>
                            </tbody>
                        </table>
                    `
                }
            }
        },
        updateYLabel () {
            return this.getMetricDisplayName(this.graphY)
        },
        updateXLabel () {
            return this.getMetricDisplayName(this.graphX)
        },
        getMetricDisplayName (metric) {
            return {
                sample: 'Sample',
                impact: 'Impact',
                power: 'Power',
                base: 'Base',
                falseposrate: 'False Positive Rate',
                sdrate: 'Base Standard deviation'
            }[metric] || ''
        }
    },
    watch: {
        sample () {
            this.updateGraphData();
        },
        impact () {
            this.updateGraphData();
        },
        power () {
            this.updateGraphData();
        },
        graphY () {
            this.updateGraphData();
        },
        graphX () {
            this.updateGraphData();
        }
    },
    mounted () {
        let {resize, createTooltip} = this,
            vueInstance = this;
        resize();

        this.dataDefault = dataDefault;

        this.chart = c3.generate({
            bindto: this.$refs['pc-graph'],
            size: {
                width: this.width,
                height: this.height
            },
            legend: {
                show: false
            },
            data: {
                x: 'x',
                columns: this.dataDefault,
                type: 'line'
            },
            axis: {
                x: {
                    label:  this.updateXLabel(),
                    tick: {
                        values (minMax) {
                            let [min, max] = minMax.map((num) => {return window.parseInt(num)}),
                                amount = 5,
                                ratio = (max - min) / amount,
                                result = new Array(amount + 1);

                            // create the values
                            result = Array.from(result).map((undef, i) => {
                                return (min + (ratio * i)).toFixed(2)
                            })

                            return result
                        },
                        format (x) {
                            let {
                                    graphX,
                                    displayValue,
                                } = vueInstance,
                                result = x;

                            if (graphX == 'sample') {
                                result = displayValue('sample', result)
                                if (result >= 1000) {
                                    result = window.parseInt(result / 1000) + 'k'
                                }
                            } else if (graphX == 'impact') {
                                result = displayValue('impact', result)
                                result += '%';
                            }

                            return result
                        }
                    }
                },
                y: {
                    label: this.updateYLabel(),
                    tick: {
                        values () {
                            let {
                                    graphY,
                                } = vueInstance,
                                result = [0, 25, 50, 75, 100];// power

                            if (graphY == 'impact') {
                                result = [0, 10, 20];// impact
                            }

                            return result
                        },
                        format (x) {
                            let {
                                    graphY,
                                    displayValue,
                                } = vueInstance,
                                result = x;

                            result += '%';

                            return result
                        }
                    }
                }
            },
            padding: {
                right: 20
            },
            tooltip: createTooltip(this)
        });

        this.updateGraphData()
    }
}

</script>

<style>
/* graph radio styles */
.pc-graph-controls {
    display: flex;
    flex-direction: row;
}

.pc-graph-radio-label {
    position: relative;
    margin-bottom: 5px;
}

.pc-graph-radio-input {
    position: absolute;
    visibility: hidden;
}

.pc-graph-radio-text {
    display: block;
    border: 1px solid var(--dark-gray);
    border-radius: 3px;
    padding: 3px;
    color: var(--black);
}

.pc-graph-radio-selected {
    border: 1px solid var(--dark-blue);
    background: var(--light-blue);
}
</style>