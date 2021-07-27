'use strict';
// const cookieAuth = require('../../../authorization/authorization');
const q = require('q');
// const commons = require('../../dbOps/commons');
// const config = require('../../../../config/config');
const connectionPool = require('../../../../utils/database/mySQLCon').connection;
// const findOpsMongo = require('../../../rseva/mongoDbOps/findOpsMongo');
// const mongoTicketModel = require('../../../rseva/models/service_request/serviceRequest');
// let mongoose = require('mongoose');
// const mongoDb = require('../../../../utils/database/mongoDb');
// const successCodeJson = require('../../../../utils/successHandling/successCode.json');
// const { object } = require('underscore');
// const SLACalculation = require('../SLA/SLACalculation');
const SLATest = require('../SLA/SLATest');
const _ = require('underscore');
const processwf = require('./processWf');
const Constants = require('../../../../utils/commons/constants');

function getTimelineForItem(req) {
    let deferred = q.defer();
    // let srTicketNumber = req.query.srTicketNumber != null ? req.query.srTicketNumber : req.body.srTicketNumber;
    let currentState = req.body.currentState != null ? req.body.currentState : deferred.reject('improper data supplied'); //deferred.reject('improper data supplied');
    let objectId = req.body.objectId != null ? req.body.objectId : deferred.reject('improper data supplied');//deferred.reject('improper data supplied');
    let condition = req.body.condition != null ? req.body.condition : deferred.reject('improper data supplied');
    let states = req.body.states != null ? req.body.states : deferred.reject('improper data supplied');
    let Status = [];
    let nextState = [];
    let previousState = [];
    let timeline = [];
    let finalResult = [];
    try {
        getStatus(objectId)
            .then(async response => {
                Status = response;
                if (response.length == 0) { //response == null || response=='' || 
                    throw deferred.reject("No data found for the selected state model")
                }
                let payload = buildPayload(response);
                let finalstate = payload.find((i) => i.is_FinalState == 1);
                let flow = [];
                if (currentState != finalstate.currentStatus_label) {
                    flow = await processwf.generateTimeline(payload, currentState);
                }
                var LongestArrayIndex; //findig array with longest path
                if (flow.length == 1) {
                    LongestArrayIndex = 0;
                }
                else if (flow.length >= 2) {
                    LongestArrayIndex = flow.reduce((acc, arr, idx) => {
                        // console.log(acc, idx, JSON.stringify([arr, arrayList[acc]]))
                        return arr.length > flow[acc].length ? idx : acc
                    }, 0)
                    //console.log("longest array is at index: ", flow[LongestArrayIndex]);
                }
                console.log("longest array is at index: ", flow[LongestArrayIndex]);
                let reInitiateState = payload.find((i) => i.currentStatus_label == finalstate.previousStatus_label);
                let reInitiate = reInitiateState.nextStatus_label.split(',') //finding the reInitiate State
                for (let i of reInitiate) {
                    if (i != finalstate.currentStatus_label) {
                        reInitiateState = i;
                    } else { reInitiateState = null }

                }
                console.log("states", states);
                let SLAStatus = [...states] //
                let buildFlowForSLA = buildSLAPayload(flow[LongestArrayIndex], SLAStatus, reInitiateState); //reInitiateState added to check if there is reInitiating flow for SLA
                // console.log("flow", buildFlowForSLA);
                // let statusTimeline = [...buildFlowForSLA];
                buildFlowForSLA.pop();
                let SLAPayload = {
                    state_parent_id: objectId,
                    state_flow: [...buildFlowForSLA],
                    condition: condition
                }
                // console.log(SLAPayload);
                let SLAResponse = await SLATest.SLACalculationFunc(SLAPayload);
                // console.log(SLAResponse);
                console.log("states", states);
                let time = buildTimeline(flow[LongestArrayIndex], states);
                for (let i of time) {  //statusTimeline
                    let color = (i.start_date == null) ? Constants.COLOR_CODE.GREY : Constants.COLOR_CODE.GREEN;
                    let completed = (i.start_date == null) ? false : true;
                    let s = completed == true ? Constants.TIMELINE_STATUS.DONE : Constants.TIMELINE_STATUS.EDIT;
                    timeline.push({
                        State: i.Start_Condition,
                        createdDate: i.start_date,
                        color: color,
                        completed: completed,
                        state: s,
                    })
                }
                for (let i of SLAResponse) {
                    let temp = [];
                    // let index = timeline.findIndex(x => x.State === `${i.End_Condition}`);
                    // let index = timeline.lastIndexOf(x => x.State === `${i.End_Condition}`);
                    var lastIndex = timeline.slice().reverse().findIndex(x => x.State === i.End_Condition);
                    var count = timeline.length - 1;
                    var index = lastIndex >= 0 ? count - lastIndex : lastIndex;
                    let color = getColor(i.SLAStatus);
                    temp = {
                        name: i.name,
                        Start_Condition: i.Start_Condition,
                        End_Condition: i.End_Condition,
                        actualDate: i.startDate,
                        endDate: i.endDate,
                        targetDate: i.SLATargetDate,
                        SLAStatus: i.SLAStatus,
                        color: color
                    }
                    timeline.splice(index, 0, temp);
                }

                let state = response.find((j) => j.current_status_label == currentState);
                let finalState = response.find((j) => j.isFinalState == 1);
                let resolveState = finalState.previous_status_label.includes(`${state.current_status_label}`) == true ? true : false;
                if (state.current_status_label != finalstate.currentStatus_label) {
                    nextState.push({
                        State: state.current_status_label,
                        id: state.object_id,
                        resolveState: resolveState
                    })
                }
                let tempnextState = state.next_status_label != '' ? state.next_status_label.split(',') : [];
                let temppreviousState = state.previous_status_label != '' ? state.previous_status_label.split(',') : [];

                for (let i of tempnextState) {
                    if (i != finalstate.currentStatus_label) { //To restrict pushing final state into next states array
                        let id = response.find((j) => j.current_status_label == i);
                        resolveState = finalState.previous_status_label.includes(`${i}`) == true ? true : false;
                        nextState.push({
                            State: i,
                            id: id.object_id,
                            resolveState: resolveState
                        })
                        // nextState.push(tempNextStatus);
                    }
                }
                // if(tempnextState.length>0) nextState.push(tempNextStatus);
                for (let i of temppreviousState) {
                    let id = response.find((k) => k.current_status_label == i);
                    previousState.push({
                        State: i,
                        id: id.object_id
                    });
                }

                finalResult.push({
                    Timeline: timeline,
                    // SLA: SLA,
                    previousState: previousState,
                    nextState: nextState,
                })
                deferred.resolve(finalResult);
            }).catch((error) => {
                console.log('error', error)
                deferred.reject(error);
            });
    } catch (error) {
        console.log('error', error)
        deferred.reject(error);
    }
    return deferred.promise
}

function buildTimeline(flows, achievedStates) {
    let temp = [];
    for (let i = 0; i < achievedStates.length; i++) { //achievedStates.length -1
        temp.push({
            Start_Condition: achievedStates[i].State,
            start_date: achievedStates[i].createdDate,
        })
    }
    for (let i in flows) {
        // let nextvar = parseInt(i) + 1;
        // let end = flows[nextvar] ? flows[nextvar] : null;
        if (i == 0) {
            // let a = temp[j].Start_Condition.includes(`${flow[0][1]}`)
            let a = temp.find((k) => k.Start_Condition == `${flows[parseInt(i)]}`);
            if (a == undefined) {
                temp.push({
                    Start_Condition: flows[parseInt(i)],
                    // End_Condition: end,
                    start_date: null,
                    // end_date: null //endD
                })
            } else { continue }
        } else {
            // let a = temp[j].Start_Condition.includes(`${flow[0][i]}`)
            // let a = temp.find((k) => k.Start_Condition == `${flow[parseInt(i)]}`);
            // if (a == null || a == undefined) {
            temp.push({
                Start_Condition: flows[parseInt(i)],
                // End_Condition: end,
                start_date: null,
                // end_date: null //endD
            })
        }

        // }
    }
    return temp
}

// function stateReduce(temp, achievedStates) {
//     // let ind=[];
//     temp.reduce(function (ind, el, i) {
//         if (el === `${achievedStates}`)
//             ind.push(i);
//         return ind;
//     }, []);
// }

function buildPayload(payload) {
    let finalResponse = [];
    for (let i of payload) {
        let finalObject = {};
        Object.assign(finalObject, {
            previousStatus_label: i.previous_status_label,
            currentStatus_label: i.current_status_label,
            nextStatus_label: i.next_status_label,
            isInital_State: i.isInitalState,
            is_FinalState: i.isFinalState
        });
        finalResponse.push(finalObject);
    }
    return finalResponse;
}

function getColor(SLAStatus) {
    if (SLAStatus == 'INSLA') {
        return Constants.COLOR_CODE.GREEN;
    } else if (SLAStatus == 'OUTSLA') {
        return Constants.COLOR_CODE.RED;
    } else if (SLAStatus == 'NA') {
        return Constants.COLOR_CODE.GREY;
    }
}

function buildSLAPayload(flow, states, reInitiateState) {
    let temp = []
    // let counters = [];
    // for (let i in flow) {
    //     let counter = 0;
    //     for (let j in states) {
    //         if (flow[i].includes(`${states[j].currentState}`)) {
    //             counter = counter + 1;
    //         }
    //     } counters.push(counter);//pushing count of similar elements
    // }
    // const max = Math.max(...counters);//finding largest counter element from counters array
    // const List = [];
    // counters.forEach((item, index) => item === max ? List.push(index) : null);//pushing the largest element's index into list array
    // let arrayList = [];
    // for (let i of List) {
    //     arrayList.push(flow[i]);//pushing the arrays with largest matched elements in arrayList
    // }
    // // let flows = [];
    // var LongestArrayIndex; //findig array with longest path
    // if (flow.length == 1) {
    //     LongestArrayIndex = 0;
    // }
    // else if (flow.length >= 2) {
    //     var LongestArrayIndex = flow.reduce((acc, arr, idx) => {
    //         // console.log(acc, idx, JSON.stringify([arr, arrayList[acc]]))
    //         return arr.length > flow[acc].length ? idx : acc
    //     }, 0)
    //     // console.log("longest array is at index: ", LongestIndex);
    // } //console.log("LongestIndex", LongestArrayIndex);

    // for (let i in flow[LongestArrayIndex]) {
    //     let j = parseInt(i) + 1;
    //     let end = flow[LongestArrayIndex][j] ? flow[LongestArrayIndex][j] : null;
    //     temp.push({
    //         Start_Condition: flow[LongestArrayIndex][i],
    //         End_Condition: end,
    //         start_date: null,
    //         end_date: null
    //     })
    // }
    // let checkDuplicate = []
    // for (let i = 0; i < states.length - 1; i++) {
    //     // let endDt = states[j] ? states[j].createdDate : null;
    //     checkDuplicate.push(states[i].State)
    // }
    // for (let i in flow) {
    //     checkDuplicate.push(flow[parseInt(i)])
    // }
    // // let unique = checkIfArrayIsUnique(checkDuplicate);
    // let duplicate = checkDuplicate.some((val, i) => checkDuplicate.indexOf(val) !== i);

    //comment this if not required
    // if(reInitiateState)
    // for(let i of states){
    //     if (states[i].Start_Condition.includes(`${reInitiateState}`)) {
    let restartFlowState = states.findIndex(x => x.State === reInitiateState)
    if (restartFlowState == -1) {
        //push normal flow here
        for (let i = 0; i < states.length; i++) { //- 1
            // if(unique==false){
            let j = parseInt(i) + 1;
            let end = states[j] ? states[j].State : null;
            let endDt = states[j] ? states[j].createdDate : null;
            temp.push({
                Start_Condition: states[i].State,
                End_Condition: end,
                start_date: states[i].createdDate,
                end_date: endDt
            })
        }

    } else {
        //discard the states before reInitial state and push into array
        // let slaStatus = states;
        let arr = states.splice(restartFlowState);
        for (let i in arr) {
            // for (let i = 0; i < arr.length - 1; i++) {
            // if(unique==false){
            let j = parseInt(i) + 1;
            let end = arr[j] ? arr[j].State : null;
            let endDt = arr[j] ? arr[j].createdDate : null;
            temp.push({
                Start_Condition: arr[i].State,
                End_Condition: end,
                start_date: arr[i].createdDate,
                end_date: endDt // null
            })
            // }

        }
    }
    //instead of checking duplicate check finalState-1 state's next state . in case of multiple conditions select the one which is not final state and get the flow from that state  

    // for (let i in states) {
    // if (duplicate == false) {
    //     for (let i = 0; i < states.length - 1; i++) {
    //         // if(unique==false){
    //         let j = parseInt(i) + 1;
    //         let end = states[j] ? states[j].State : null;
    //         // let endDt = states[j] ? states[j].createdDate : null;
    //         temp.push({
    //             Start_Condition: states[i].State,
    //             End_Condition: end,
    //             start_date: states[i].createdDate,
    //             end_date: null //endDt
    //         })
    //     }
    // }

    // for (let i in flow[LongestArrayIndex]) {
    //     let nextvar = parseInt(i) + 1;
    //     let end = flow[LongestArrayIndex][nextvar] ? flow[LongestArrayIndex][nextvar] : null;
    //     // let a = temp[j].Start_Condition.includes(`${flow[0][i]}`)
    //     let a = temp.find((k) => k.Start_Condition == `${flow[LongestArrayIndex][parseInt(i)]}`);
    //     if (a == null || a == undefined) {
    //         temp.push({
    //             Start_Condition: flow[LongestArrayIndex][parseInt(i)],
    //             End_Condition: end,
    //             start_date: null,
    //             end_date: null //endD
    //         })
    //     }
    // }

    for (let i in flow) {
        let nextvar = parseInt(i) + 1;
        let end = flow[nextvar] ? flow[nextvar] : null;
        // let a = temp[j].Start_Condition.includes(`${flow[0][i]}`)
        // let a = temp.find((k) => k.Start_Condition == `${flow[parseInt(i)]}`);
        let a = temp.findIndex(x => x.Start_Condition === flow[parseInt(i)])
        if (a == -1) { //a == undefined
            temp.push({
                Start_Condition: flow[parseInt(i)],
                End_Condition: end,
                start_date: null,
                end_date: null //endD
            })
        } else if (a == 1) {
            Object.assign(temp[a], {
                End_Condition: end
            })
        }
    }

    // if (!duplicate) {
    // for (let i in temp) {
    //     for (let j in states) {
    //         if (temp[i].Start_Condition.includes(`${states[j].State}`)) {//.toLowerCase()
    //             Object.assign(temp[i], {
    //                 start_date: states[j].createdDate,
    //                 end_date: (states[parseInt(j) + 1] && states[parseInt(j) + 1].createdDate) ? states[parseInt(j) + 1].createdDate : null
    //             })
    //         }
    //     }
    // }
    // } 
    // else {
    //     //if duplicate then check if states has multiple value .if states has multiple values then push last occuring date else dont push the date
    //     // let lastDuplicate=[];
    //     // for (let j in states) {
    //     //     let first = states.findIndex(x => x.State === states[j].State)
    //     //     // let last = states.lastIndexOf(item => item.State === `${states[j].State}`)
    //     //     var last = states.slice().reverse().findIndex(x => x.State === states[j].State);
    //     //     var count = states.length - 1;
    //     //     var finalIndex = last >= 0 ? count - last : last;
    //     //     // timeline.lastIndexOf(x => x.State === `${i.End_Condition}`);
    //     //     //var index = array.slice().reverse().findIndex(x => x[searchKey] === searchValue);
    //     //     console.log("first",first,last,states[j].State,finalIndex);
    //     // }
    //     for (let i in temp) {
    //         for (let j in states) {
    //             let first = states.findIndex(x => x.State === states[j].State)
    //             // let last = states.lastIndexOf(item => item.State === `${states[j].State}`)
    //             var lastIndex  = states.slice().reverse().findIndex(x => x.State === states[j].State);
    //             var count = states.length - 1;
    //             var last = lastIndex >= 0 ? count - lastIndex : lastIndex;
    //             // if(first!=last && j==first){
    //             //     lastDuplicate.push(states[last])
    //             // }
    //             // states[j].State.includes()
    //             if (temp[i].Start_Condition==states[j].State && first!=last && j==first) {//.toLowerCase()
    //                 // temp[i].start_date=states[last].createdDate
    //                 // temp[i].end_date=states[parseInt(last) + 1]
    //                 Object.assign(temp[i], {
    //                     start_date: states[last].createdDate,
    //                     end_date: (states[parseInt(last) + 1] && states[parseInt(last) + 1].createdDate) ? states[parseInt(last) + 1].createdDate : null
    //                 })
    //             }
    //         }
    //     }
    // }

    console.log(temp);
    // temp.pop(); //To remove the last element of the array 
    return temp
}

function getStatus(objectId) {

    let deferred = q.defer();
    // let objectId = req.query.objectId != null ? req.query.objectId : req.body.objectId; //config.utils.incident.processModelId
    //id, fields, display, hide, mandatory, optional, alternate_ field_ label
    try {
        let QueryParam = `SELECT object_id, current_status_label, previous_status_label, next_status_label, isInitalState, isFinalState FROM RCMFOSA.tbl_itsm_processmodel_state_flow where state_parent_id='${objectId}' ORDER BY stateSequence`; //dbCommons.SELECT_QUERY.QUERY_FETCH_PROCESS_MODEL_DETAILS
        connectionPool.getConnection((error, connection) => {
            if (error) {
                deferred.reject(error)
            } else {
                connection.query(QueryParam, function (error, rows) {
                    if (error) {
                        console.log('NoDataFound Error', error);
                        deferred.reject(error)
                    } else {
                        deferred.resolve(rows)
                    }
                })
            }
        })
        // .catch(error => {
        //     console.log(error);
        //     deferred.reject(error);
        // })
    } catch (error) {
        deferred.reject(error);
    }
    return deferred.promise

}

// module.exports.getStates = getStates;
module.exports.getTimelineForItem = getTimelineForItem;
// module.exports.callFindStatus = callFindStatus;
// module.exports.callFindStatusForReference = callFindStatusForReference;
module.exports.buildPayload = buildPayload;
