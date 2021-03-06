<template>
    <div ref="container" class="flex flex-1 items-start align-middle ml-12 px-2 pt-4 bg-gray-600 text-gray-800">
        <div id="defaultEventsContainer" ref="defaultEventsContainer" :style="this.editCalendar ? '' : 'border-color: transparent'" class="flex justify-center flex-col border border-black bg-gray-600 mr-3 mt-20 p-1 h-auto">
            <div class="flex justify-center items-center w-32">
                <input class="p-1 hidden" type="checkbox" name="editCheckbox" v-on:change="styleEditCheckboxLabel" id="editCheckbox" v-model="editCalendar">
                <label class="text-gray-800" id="editCheckboxLabel" for="editCheckbox">Modifier</label>
            </div>
            <transition name="edit-menu">
                <div v-if="editCalendar">
                    <div class="border border-black p-1 m-1">
                        <h1 class="block uppercase tracking-wide text-gray-800 text-xs font-bold mb-2">Activités de base</h1>
                        <div v-for="defaultEvent in getDefaultEvents" class='draggable-event fc-event p-1 my-1' :style="getEventBackgroundColor(defaultEvent.title)" :data-event="getEventData(defaultEvent.title)" :key="defaultEvent.title">{{defaultEvent.title}}</div>
                    </div>

                    <div class="border border-black p-1 m-1">
                        <label class="block uppercase tracking-wide text-gray-800 text-xs font-bold mb-2" for="slotDuration">Intervalle de temps</label>
                        <select v-model="slotDuration" name="slotDuration" id="slotDuration">
                            <option v-for="slotDuration in slotDurations" :value="slotDuration.value" :key="slotDuration.value">{{ slotDuration.text }}</option>
                        </select>
                    </div>
                </div>
            </transition>
        </div>
        <FullCalendar id="calendar" ref="calendar"
                      v-resize="changeCalendarHeight"
                      defaultView="weekTimeGridView"
                      :default-date="firstDay"
                      :height="calendarHeight"
                      :plugins="calendarPlugins"
                      :locale="locale"
                      all-day-text="Chef de jour"
                      min-time="06:00:00"
                      max-time="23:30:00"
                      nav-links="true"
                      :editable="editCalendar"
                      selectable="true"
                      droppable="true"
                      event-limit="true"
                      :slot-duration="slotDuration"
                      :snap-duration="minSlotDuration"
                      force-event-duration="true"
                      v-on:select="selectionChanged"
                      v-on:dateClick="dateChanged"
                      v-on:eventReceive="eventDropped"
                      v-on:eventDrop="updateEventPosition"
                      v-on:eventResize="updateEventDuration"
                      v-on:eventClick="eventClicked"
                      :slot-label-format="slotLabelFormat"
                      :custom-buttons="getCustomButtons()"
                      :header="{
                        center: 'addEventButton, clearAllEvents',
                        right: 'weekTimeGridView, timeGridDay'
                        }"
                      :views="{ weekTimeGridView }"
                      class="z-10"
        />
        <AddEventModal ref="addEventModal" v-bind:edit-event="editCalendar" v-bind:edit-duration="editModalDuration" v-bind:is-default-event="activeEvent.isDefault" v-bind:prop-title="activeEvent.title" v-bind:prop-all-day="activeEvent.allDay" v-bind:prop-managers="activeEvent.managers" v-bind:prop-type="activeEvent.type" v-bind:add-event="modalAddEvent"
                       v-if="showModal" v-on:close="clearModalDialog" v-on:submit="addModalEvent" v-on:delete="deleteModalEvent" v-on:edit="editModalEvent"></AddEventModal>
    </div>
</template>

<script>
    import resize from 'vue-resize-directive'

    import FullCalendar from '@fullcalendar/vue';
    import dayGridPlugin from '@fullcalendar/daygrid';
    import timeGridPlugin from '@fullcalendar/timegrid';
    import momentPlugin from '@fullcalendar/moment';
    import { toMoment, toDuration } from '@fullcalendar/moment';
    import interactionPlugin, { Draggable } from '@fullcalendar/interaction';
    import frLocale from '../assets/js/cm-fr';
    import AddEventModal from "../components/EventModal";

    import { mapGetters } from 'vuex';
    import { Manager } from "../store/modules/events";

    const { ipcRenderer } = require('electron');
    let moment = require('moment');
    let cloneDeep = require('lodash.clonedeep');


    export default {
        name: 'home',
        directives: {
            resize
        },
        components: {
            AddEventModal,
            FullCalendar
        },
        computed: mapGetters(['getDefaultEvents', 'getManagers', 'getEventTypes', 'getEventColor']),
        data() {
            return {
                calendarPlugins: [ dayGridPlugin, timeGridPlugin, interactionPlugin, momentPlugin ],
                locale: frLocale,
                slotLabelFormat: {
                    hour: 'numeric',
                    minute: '2-digit',
                    omitZeroMinute: false,
                    meridiem: 'short'
                },
                weekTimeGridView: {
                    type: 'timeGrid',
                    duration: { days: this.campDuration},
                    buttonText: 'Semaine'
                },
                calendarHeight: this.getCalendarHeight(),
                selectionInfo: null,
                selectedDate: null,
                calendar: null,
                showModal: false,
                editModalDuration: true,
                modalAddEvent: false,
                activeEvent: {
                    id: '',
                    allDay: false,
                    title: '',
                    managers: [Manager],
                    type: String,
                    isDefault: false
                },
                editCalendar: false,
                slotDurations: [
                    { value: "00:15:00", text: "15 minutes" },
                    { value: "00:30:00", text: "30 minutes" },
                    { value: "01:00:00", text: "1 heure" }
                ],
                slotDuration: "00:30:00",
                minSlotDuration: "00:15:00",
                eventsIds: []
            }
        },
        props: {
            firstDay: String,
            campDuration: Number
        },
        created() {
            this.weekTimeGridView.duration.days = 6;
            this.firstDay = '2019-07-15';
        },
        mounted() {
            this.$nextTick(() => {
                this.calendar = this.$refs.calendar.getApi();

                ipcRenderer.on('async-response-all-events', (ev, arg) => {
                    for(let event of arg){
                        delete event['_id']; // Delete the given NeDB id
                        this.calendar.addEvent(event);
                        this.eventsIds.push(event.id);
                    }
                });

                /*   Makes all the default events draggable in the calendar   */
                let container = document.getElementById('defaultEventsContainer');
                new Draggable(container, {
                    itemSelector: '.draggable-event'
                });

                ipcRenderer.send('async-request-all-events');
            });
        },
        methods: {
            /* Retrieves the calendar current height */
            getCalendarHeight: function () {
                if(this.$refs.container !== undefined) {
                    return this.$refs.container.clientHeight - 32;
                }
                else {
                    return 0;
                }
            },
            /* Changes the calendar's height according to the current computed height */
            changeCalendarHeight: function () {
                this.calendarHeight = this.getCalendarHeight();
            },
            getCustomButtons: function() {
                if(this.editCalendar){
                    return {
                        addEventButton: {
                            text: "Nouvelle activité",
                            click: this.showModalDialog
                        },
                        clearAllEvents: {
                            text: "Effacer tout",
                            click: this.clearAllEvents
                        }
                    };
                }
            },
            styleEditCheckboxLabel: function() {
                let editCheckbox = document.getElementById('editCheckbox');
                let editCheckboxLabel = document.getElementById('editCheckboxLabel');
                if(editCheckbox.checked){
                    editCheckboxLabel.setAttribute('class', 'bg-gray-500 text-white');
                }
                else {
                    editCheckboxLabel.setAttribute('class', 'text-gray-800');
                }
            },
            /* Stores the current selection */
            selectionChanged: function (selectionInfo) {
                this.selectionInfo = selectionInfo;
                this.selectedDate = null;
            },
            /* Stores the current selected dateTime */
            dateChanged: function(dateInfo) {
                this.selectionInfo = null;
                this.selectedDate = dateInfo;
                this.showModalDialog({
                    start: dateInfo.date,
                    allDay: dateInfo.allDay
                });
            },
            eventDropped: function(info){
                this.selectionInfo = null;
                this.selectedDate = null;

                ipcRenderer.send('async-new-event', this.normalizeEventObject(info.event));
            },
            updateEventPosition: function(eventDropInfo){
                if(eventDropInfo.oldEvent.allDay !== eventDropInfo.event.allDay){
                    eventDropInfo.revert();
                }
                else {
                    ipcRenderer.send('async-replace-event', {
                        old: this.normalizeEventObject(eventDropInfo.oldEvent),
                        new: this.normalizeEventObject(eventDropInfo.event)
                    });
                    console.log('Sent moved event');
                }
            },
            updateEventDuration: function(eventResizeInfo){
                ipcRenderer.send('async-replace-event', { old: this.normalizeEventObject(eventResizeInfo.prevEvent), new: this.normalizeEventObject(eventResizeInfo.event)});
                console.log('Sent resized event');
            },
            showModalDialog: function(eventInfo){
                if(this.editCalendar === true) {
                    this.activeEvent.allDay = eventInfo.allDay;
                    this.editModalDuration = !(eventInfo.allDay || this.selectionInfo !== null); // If it's an all day event or a selection, do not edit the duration
                    this.modalAddEvent = true;
                    this.showModal = true;
                }
            },
            eventClicked: function(eventInfo) {
                this.editModalDuration = false;
                this.modalAddEvent = false;
                this.activeEvent.id = eventInfo.event.id;
                this.activeEvent.title = eventInfo.event.title;
                this.activeEvent.allDay = eventInfo.event.allDay;
                this.activeEvent.managers = this.getEventManagers(eventInfo.event.extendedProps.managers);
                this.activeEvent.type = eventInfo.event.extendedProps.type;
                this.activeEvent.isDefault = eventInfo.event.extendedProps.isDefaultEvent === true;
                this.showModal = true;
            },
            addModalEvent: function(event) {
                this.showModal = false;
                if(this.selectedDate !== null) {
                    event.start = this.selectedDate.date;
                    if (event.duration === '24:00') {
                        event.allDay = true;
                    } else {
                        let duration = toDuration(event.duration);
                        let endDate = moment(toMoment(this.selectedDate.date, this.calendar)).add(duration);
                        event.end = endDate.format();
                    }
                }
                else{
                    event.start = this.selectionInfo.start;
                    event.end = this.selectionInfo.end;
                    event.allDay = this.selectionInfo.allDay;
                }
                event.extendedProps = {
                    'managers': event.managers,
                    'type': event.type
                };
                this.newEvent(event);

                this.clearModalDialog();
            },
            editModalEvent: function(event){
                this.clearModalDialog();
                let newEvent = this.calendar.getEventById(this.activeEvent.id);
                let oldEvent = cloneDeep(newEvent);
                newEvent.setProp('title', event.title);
                newEvent.setProp('backgroundColor', this.getEventColor(event.type));
                newEvent.setProp('borderColor', this.getEventColor(event.type));
                newEvent.setExtendedProp('managers', event.managers);
                newEvent.setExtendedProp('type', event.type);
                newEvent.setExtendedProp('isDefaultEvent', event.isDefault);

                ipcRenderer.send('async-replace-event', { old: this.normalizeEventObject(oldEvent), new: this.normalizeEventObject(newEvent)});
                console.log('Sent edited event');
            },
            deleteModalEvent: function() {
                this.clearModalDialog();
                let event = this.calendar.getEventById(this.activeEvent.id);
                ipcRenderer.send('async-delete-event', this.normalizeEventObject(event));
                console.log('Sent delete event');

                event.remove();
            },
            clearModalDialog: function(){
                this.showModal = false;
                this.selectionInfo = null;
                this.selectedDate = null;
                this.activeEvent.isDefault = false;
            },
            clearAllEvents: function(){
                if(this.calendar !== undefined) {
                    if(this.calendar.getEvents() !== undefined) {
                        this.calendar.getEvents().forEach(e => e.remove());
                        ipcRenderer.send('async-clear-all-events');
                    }
                }
            },
            getUniqueId: function() {
                let id;
                do {
                    id = Math.random().toString(36).substr(2, 16);
                } while (this.eventsIds.indexOf(id) !== -1);
                this.eventsIds.push(id);
                return id;
            },
            /* Creates a new event from the selection data */
            newEvent: function (eventInfo) {
                let event = {};
                if(this.selectionInfo !== null){
                    console.log('Adding selection event');
                    event = {
                        id: this.selectionInfo.id,
                        allDay: this.selectionInfo.allDay,
                        start: this.selectionInfo.start,
                        end: this.selectionInfo.end,
                        title: eventInfo.title,
                        color: eventInfo.color,
                        extendedProps: eventInfo.extendedProps
                    };
                }
                else if(eventInfo !== undefined){
                    console.log('Adding eventInfo');
                    event = {
                        id: eventInfo.id,
                        start: eventInfo.start,
                        end: eventInfo.end,
                        title: eventInfo.title,
                        color: eventInfo.color,
                        allDay: eventInfo.allDay,
                        extendedProps: eventInfo.extendedProps
                    };
                }
                event.id = this.getUniqueId();
                let normalizedObject = this.normalizeEventObject(event);
                console.log(normalizedObject);
                let generatedEvent = this.calendar.addEvent(normalizedObject);
                console.log(generatedEvent);
                ipcRenderer.send('async-new-event', this.normalizeEventObject(generatedEvent));
            },
            /* Get the data-event of the corresponding defaultEvent */
            getEventData: function (title) {
                let defaultEvent = this.getDefaultEvents.find(e => e.title === title);
                return JSON.stringify(defaultEvent);
            },
            getEventBackgroundColor: function(title){
                let defaultEvent = this.getDefaultEvents.find(e => e.title === title);
                let defaultEventColor = this.getEventColor(defaultEvent.extendedProps.type);
                return 'background-color: ' + defaultEventColor + '; border-color: ' + defaultEventColor + ';';
            },
            getEventManagers: function(managers) {
                let newManagers = [];
                for(let m of managers) {
                    newManagers.push(new Manager(m.code, m.fullName));
                }
                return newManagers;
            },
            normalizeEventObject: function (EventObject) {
                return {
                    id: EventObject.id,
                    start: EventObject.start,
                    end: EventObject.end,
                    title: EventObject.title,
                    allDay: EventObject.allDay,
                    backgroundColor: EventObject.backgroundColor !== (undefined && '') ? EventObject.backgroundColor : this.getEventColor(EventObject.extendedProps.type),
                    borderColor: EventObject.borderColor !== (undefined && '') ? EventObject.borderColor : this.getEventColor(EventObject.extendedProps.type),
                    extendedProps: EventObject.extendedProps
                };
            }
        }
    }
</script>

<style scoped>
    @import "~@fullcalendar/core/main.css";
    @import "~@fullcalendar/daygrid/main.css";
    @import "~@fullcalendar/timegrid/main.css";

    ::-webkit-scrollbar {
        background-color: rgba(0, 0, 0, 0);
    }

    .draggable-event {
        @apply bg-blue-500 cursor-move text-center;
    }

    .draggable-event:hover {
        @apply bg-blue-400;
    }

    #editCheckboxLabel {
        @apply px-1 text-lg block uppercase tracking-wide font-bold rounded;
    }

    #editCheckboxLabel:hover {
        @apply bg-gray-500;
    }

    .edit-menu-enter,
    .edit-menu-leave-active {
        opacity: 0;
    }

    .edit-menu-enter-active,
    .edit-menu-leave-active {
        transition: opacity .5s ease;
    }
</style>