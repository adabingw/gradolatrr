<script>
    import { createEventDispatcher } from "svelte";
    import { mutation } from "svelte-apollo";

    import { UPDATE_ASSIGNMENT, UPDATE_COURSE, UPDATE_TERM } from "../constants/queries_put";
    import { sortOrder, dragstart, drop, maxOrder } from "./utils.svelte";
    import TextArea from "./TextArea.svelte";
    import TextField from "./TextField.svelte";
    import Properties from "./Properties.svelte";
    import NewProperty from "./NewProperty.svelte";
    import Datepicker from "./Datepicker.svelte";
    import ContextMenu from "./ContextMenu.svelte";
    import DateComp from "./Date.svelte";
    import Multiselect from "./Multiselect.svelte";

    export let cmd;
    export let info;

    let data = info["data"];
    data = JSON.parse(data);
        
    let content_info;
    if (cmd == "course") {
        content_info = JSON.parse(info["content_info"]);
    }

    let data_array = [];
    for (const key in data) {
        data_array.push([ key, data[key] ])
    }
    data_array = sortOrder(data_array);

    let showMenu = false;
    let context_bundle = [ 0, 0, 0 ];
    let update;

    if (cmd == "term") update = mutation(UPDATE_TERM);
    else if (cmd == "course") update = mutation(UPDATE_COURSE);
    else if (cmd == "assign") update = mutation(UPDATE_ASSIGNMENT);

    const dispatch = createEventDispatcher();

    function dragover (ev, i) {
        ev.preventDefault();
        ev.dataTransfer.dropEffect = 'move';

        let target = document.getElementById(`${data_array[i][0]}`);
        if (target) {
            target.style.borderTop = '1px solid blue';
        }
    }

    function dragleave (ev, i) {
        ev.preventDefault();
        ev.dataTransfer.dropEffect = 'move';

        let target = document.getElementById(`${data_array[i][0]}`);
        if (target) {
            target.style.borderTop = '0px solid blue';
        }
    }

    async function dropEvent(ev, key2, index2) {
        let target = document.getElementById(`${data_array[index2][0]}`);
        if (target) {
            target.style.borderTop = '0px solid blue';
        }

        let return_info = drop(ev, key2, index2, data_array, info);
        if (!return_info) return;

        info = return_info[0];
        data_array = return_info[1];
        data_array = sortOrder(data_array);

        if (cmd == "term") {
            try {
                await update({
                    variables: {
                        input: {
                            id: info["id"], 
                            type: "term", 
                            data: info["data"]
                        }
                    }
                });
            } catch(error) {
                console.error(error);
            }
        } else if (cmd == "course") {
            try {
                await update({
                    variables: {
                        input: {
                            id: info["id"], 
                            type: "course", 
                            data: info["data"]
                        }
                    }
                });
            } catch(error) {
                console.error(error);
            }
        } else if (cmd == "assign") {
            try {
                await update({
                    variables: {
                        input: {
                            id: info["id"], 
                            type: "item", 
                            data: info["data"]
                        }
                    }
                });
            } catch(error) {
                console.error(error);
            }
        }
    }

    function infoController(event) {
        if (event.detail.info == "delete") {            
            delete content_info[event.detail.data]
            dispatch('action', {
                action: 'delete',
                info_name: event.detail.data, 
            })
        } else if (event.detail.info == "saved") {
            content_info[event.detail.info_name] = event.detail.new_info;
            dispatch('action', {
                'action': 'saved',
                info_name: event.detail.info_name, 
                new_info: event.detail.new_info['type']
            })
        } else if (event.detail.info == "tags") {
            content_info[event.detail.info_name] = event.detail.tags;
        } else if (event.detail.info == "type") {
            content_info[event.detail.info_name] = event.detail.new_info;
        }
        info["content_info"] = JSON.stringify(content_info);
        content_info = content_info;
    }

    function addedProperty(event) {
        const new_name = event.detail.name;
        const new_type = event.detail.type; 
        
        let new_property = {
            "type": new_type, 
            "order": (cmd == "assign" || cmd == "bundle") ? 1 : maxOrder(data_array) + 1
        }

        if (new_type == "text" || new_type == "textarea") new_property["content"] = "";
        else if (new_type == "number") new_property["content"] = 0;
        else if (new_type == "multiselect" || new_type == "singleselect") new_property["content"] = [["", 0]];

        data_array.push([new_name, new_property]);
        data[new_name] = new_property;
        info["data"] = JSON.stringify(data);
        data_array = data_array;
        let data_temp = JSON.stringify(data);
        dispatch('message', {
            data: data_temp
        });
    }

    function dataChange() {
        for (const key of data_array) {
            data[key[0]] = key[1];
        }
        let data_temp = JSON.stringify(data);
        dispatch('message', {
            data: data_temp
        });
    }

    function dataChangeSelect(event, key) {
        dispatch('message', {
            data: data, 
            key: key
        });
    }

    function openMenu(e, index, item) {
        showMenu = false;
        e.preventDefault();
        context_bundle = [e.clientX, e.clientY, index, item];
        showMenu = true;
    }

    function contextController(e) {
        const context = e.detail.context; 
        const subcontext = e.detail.subcontext;
        const index = e.detail.index;
        const item = e.detail.item;
            
        if (context == "change_type" && subcontext) {
            if (subcontext == data[item]['type']) return;
            data[item]['type'] = subcontext;
            if (subcontext == "number") {
                data[item]['content'] = parseInt(data[item]['content'])
            } else if (subcontext == "date") {
                data[item]['content'] = (new Date()).toISOString().split('T')[0]
            } else {
                data[item]['content'] = data[item]['content'].toString();
            }
            info["data"] = JSON.stringify(data);
        } else {
            data_array.splice(index, 1);
            delete data[item];
            info["data"] = JSON.stringify(data);
            data_array = data_array;
        }
    }

    function updateInfo() {     
        if (typeof info["data"] != 'object') data = JSON.parse(info["data"]);
        if (data_array.length != 0) data_array = [];
        for (const key in data) {
            data_array.push([ key, data[key] ])
        }
        data_array = sortOrder(data_array);
    }

    $: {
        info;
        updateInfo();
    }

</script>

<ContextMenu bind:showMenu={showMenu} 
        bind:x={context_bundle[0]} 
        bind:y={context_bundle[1]} 
        bind:index={context_bundle[2]}
        bind:item={context_bundle[3]}
        menuNum={1}
        on:context={contextController}/>
<table class="Table">
<div class="Table">
    <tbody>
        {#if data_array.length > 0 && data_array != undefined}
            {#each data_array as data, i}
            {#if data[0] != "name"}
                <tr id={`${data[0]}`}>
                <!-- svelte-ignore a11y-no-static-element-interactions -->
                <div class="TableBodyRow"
                    draggable={true}
                    on:dragstart={event => dragstart(event, data[0] , i)}
                    on:drop={event => dropEvent(event, data[0], i)} on:dragover={(event) => dragover(event, i)}
                    on:dragleave={(event) => dragleave(event, i)}
                    >
                    <td>
                        <span class="bodycellheader tablecol">
                        {#if cmd != "assign" && cmd != "bundle"}
                            <i class="fa-solid fa-ellipsis-vertical context_menu" 
                            on:click={(e) => { e.stopPropagation(); openMenu(e, i, data[0])}}></i>
                        {/if}
                        {#if data[1]["type"] == "textarea"}
                            <i class="fa-solid fa-align-justify component"></i>
                        {:else if data[1]["type"] == "text" || data[1]["type"] == "number"}
                            {#if data[1]["type"] == "text"}
                                <i class="fa-solid fa-font component"></i>
                            {:else}
                                <i class="fa-solid fa-hashtag component"></i>
                            {/if}
                        {:else if data[1]["type"] == "multiselect" && (cmd == "assign" || cmd == "bundle")}
                            <i class="fa-solid fa-list component"></i>
                        {:else if data[1]["type"] == "singleselect" && (cmd == "assign" || cmd == "bundle")}
                            <i class="fa-regular fa-circle-check component"></i>
                        {:else if data[1]["type"] == "date"}
                            <i class="fa-regular fa-calendar component"></i>
                        {/if}
                            <p>{data[0]}</p>
                        </span>
                    </td>
                </div>
                </tr>
                <tr>
                    <div class="TableBodyRow TableBodyText">
                        <td>
                            {#if data[1]["type"] == "textarea"}
                                <TextArea bind:inputText={data[1]["content"]} on:message={dataChange}/>
                            {:else if data[1]["type"] == "text" || data[1]["type"] == "number"}
                                 <TextField bind:inputText={data[1]["content"]} 
                                    text="nothing here yet..." type={data[1]["type"]} on:message={dataChange}
                                    max={100} min={0}  focus={false}/>
                            {:else if data[1]["type"] == "multiselect" && (cmd == "assign" || cmd == "bundle")}
                                <Multiselect bind:properties={data[1]["content"]} bind:selections={data[1]["tag_info"]}
                                    on:assign={(e) => dataChangeSelect(e, data[0])} on:course={(e) => dataChangeSelect(e, data[0])} max=0/>
                            {:else if data[1]["type"] == "singleselect" && (cmd == "assign" || cmd == "bundle")}
                                <Multiselect bind:properties={data[1]["content"]} bind:selections={data[1]["tag_info"]}
                                on:assign={(e) => dataChangeSelect(e, data[0])} on:course={(e) => dataChangeSelect(e, data[0])} max=1/>
                            {:else if data[1]["type"] == "date" && cmd != "bundle"}
                                <DateComp bind:date={data[1]["content"]} on:message={dataChange} />
                            {:else if data[1]["type"] == "date" && cmd == "bundle"}
                                <span class="details">
                                    You can pick multiple days here. The days will be applied in chronological order. <br/>
                                    If you pick more than the number of items (n), only the n dates picked will be applied. <br/>
                                    If you pick less than the number of items, the last day will be applied for the remaining items.
                                </span>
                                <Datepicker bind:dates={data[1]["content"]} bind:num={data[1]["num"]} on:message={dataChange}/>
                            {/if}
                        </td>
                    </div>
                </tr>
            {/if} 
            {/each} 
        {/if}
        </tbody>
</div>
</table>
{#if cmd != "assign" && cmd != "bundle"}
    <NewProperty on:message={addedProperty} data={data_array}/>
{/if}
{#if cmd == "course"}
    <p class="subheader">Information about items in this course</p>
    <Properties bind:courseinfo={content_info} on:info={infoController}/>
{/if}

<style>
.details {
    color: #818181;
    font-size: 15px;
}

.context_menu {
    opacity: 0.0;
}

.context_menu:hover {
    opacity: 1.0;
}

.tablecol {
    width: 100%;
}

.TableBodyRow {
    display: flex; 
    align-items: center;
    width: 100%;
}

.bodycellheader {
    display: flex;
    flex-direction: row;
    align-items: center;
    width: fit-content;
    min-width: fit-content;
}

.TableBodyText {
    display: flex;
    flex-direction: row;
    align-items: center;
    margin-top: -15px;
    margin-left: 15px;
    width:100%;
}

.TableBodyRow:hover {
    cursor: pointer;
}

.Table {
    width:100%;
}

table, tbody, tr, td, .TableBodyRow {
    width: 100%;
}
</style>
