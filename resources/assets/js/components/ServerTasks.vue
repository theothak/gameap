<template>
    <div id="server-task-component">
        <button class="btn btn-success m-1" data-toggle="modal" v-on:click="createTask()">{{ trans('main.add')}}</button>
        <hr>
        <table class="table table-striped table-bordered">
            <thead>
                <tr>
                    <td>{{ trans('servers_tasks.task') }}</td>
                    <td>{{ trans('servers_tasks.date') }}</td>
                    <td>{{ trans('servers_tasks.repeat') }}</td>
                    <td>{{ trans('main.actions') }}</td>
                </tr>
            </thead>
            <tbody v-for="(value, key) in tasks">
                <tr>
                    <td>{{ value.command }}</td>
                    <td>{{ value.execute_date }}</td>
                    <td>{{ humanRepeatText(value.repeat) }}</td>
                    <td>
                        <button class="btn btn-sm btn-info btn-success m-1" v-on:click="editTask(key)">{{ trans('main.edit') }}</button>
                        <button class="btn btn-sm btn-info btn-danger m-1" v-on:click="deleteTask(key)">{{ trans('main.delete') }}</button>
                    </td>
                </tr>
            </tbody>
        </table>

        <div class="modal fade" tabindex="-1" role="dialog" id="createTaskModal" aria-hidden="true">
            <div class="modal-dialog" role="document">
                <div class="modal-content">
                    <div class="modal-header">
                        <h5 class="modal-title">{{ modalTitle }}</h5>
                        <button type="button" class="close" data-dismiss="modal" :aria-label="trans('main.close')">
                            <span aria-hidden="true">&times;</span>
                        </button>
                    </div>

                    <div class="modal-body">
                        <form>
                            <div class="form-group">
                                <label for="task" class="control-label">{{ trans('servers_tasks.task') }}</label>

                                <select
                                        id="command"
                                        class="custom-select"
                                        name="command"
                                        v-model="command"
                                        v-on:change="formChange">
                                    <option v-if="privileges.restart" value="restart">{{ trans('servers.restart') }}</option>
                                    <option v-if="privileges.start" value="start">{{ trans('servers.start') }}</option>
                                    <option v-if="privileges.stop" value="stop">{{ trans('servers.stop') }}</option>
                                    <option v-if="privileges.update" value="update">{{ trans('servers.update') }}</option>
                                    <option v-if="privileges.update" value="reinstall">{{ trans('servers.reinstall') }}</option>
                                </select>

                                <span v-if="errors['command']" class="help-block">
                                    <strong class="text-danger">{{ errors['command'] }}</strong>
                                </span>
                            </div>

                            <div class="form-group">
                                <date-picker
                                        type="datetime"
                                        v-model="taskDate"
                                        valueType="format"
                                        v-on:change="formChange">
                                </date-picker><br>

                                <span v-if="errors['taskDate']" class="help-block">
                                    <strong class="text-danger">{{ errors['taskDate'] }}</strong>
                                </span>
                            </div>

                            <div class="form-check">
                                <label class="control-label">
                                    <input
                                            v-model="taskRepeatRadio"
                                            v-on:change="formChange"
                                            type="radio"
                                            name="repeat"
                                            value="1">
                                    {{ trans('servers_tasks.no_repeat') }}
                                </label>
                            </div>

                            <div class="form-check">
                                <label class="control-label">
                                    <input
                                            v-model="taskRepeatRadio"
                                            v-on:change="formChange"
                                            type="radio"
                                            name="repeat"
                                            value="0">
                                    {{ trans('servers_tasks.endlessly_repeat') }}
                                </label>
                            </div>

                            <div class="form-check">
                                <label class="control-label">
                                    <input
                                            v-model="taskRepeatRadio"
                                            v-on:change="formChange"
                                            type="radio"
                                            name="repeat"
                                            value="">
                                    {{ trans('servers_tasks.custom_repeat') }}
                                </label>
                            </div>

                            <div class="form-group">
                                <label for="repeat" class="control-label">{{ trans('servers_tasks.repeat_num') }}</label>
                                <input
                                        v-model.number="taskRepeatInput"
                                        :disabled="taskRepeatRadio !== ''"
                                        id="repeat"
                                        type="number"
                                        min="1"
                                        max="255"
                                        class="form-control">

                                <span v-if="errors['taskRepeatInput']" class="help-block">
                                    <strong class="text-danger">{{ errors['taskRepeatInput'] }}</strong>
                                </span>
                            </div>

                            <div class="form-group">
                                <label class="control-label">{{ trans('servers_tasks.repeat_period') }}</label>

                                <div class="row">
                                    <div class="col-md-4">
                                        <input v-model="taskRepeatPeriod"
                                               :disabled="repeat === 1"
                                               v-on:change="formChange"
                                               id="repeat_period"
                                               name="repeat_period"
                                               type="number"
                                               :min="repeatMin"
                                               class="form-control">
                                    </div>

                                    <div class="col-md-8">
                                        <select
                                                v-model="taskRepeatUnit"
                                                :disabled="repeat === 1"
                                                v-on:change="formChange"
                                                class="custom-select"
                                                >
                                            <option v-for="(value, key) in unitOptions" :value="key">{{ value }}</option>
                                        </select>
                                    </div>
                                </div>

                                <span v-if="errors['taskRepeatPeriod']" class="help-block">
                                    <strong class="text-danger">{{ errors['taskRepeatPeriod'] }}</strong>
                                </span>

                            </div>

                        </form>
                    </div>

                    <div class="modal-footer">
                        <button type="button" class="btn btn-secondary" data-dismiss="modal">{{ trans('main.close') }}</button>
                        <button type="button" class="btn btn-primary" v-on:click="sendTaskForm">{{ buttonName }}</button>
                    </div>
                </div>
            </div>
        </div>
    </div>
</template>

<script>
    import DatePicker from 'vue2-datepicker';
    import { mapState } from 'vuex';

    const REPEAT_ENDLESSLY          = 0;
    const REPEAT_ONCE               = 1;

    const RADIO_REPEAT_CUSTOM       = '';

    export default {
        components: { DatePicker },
        props: {
            serverId: Number,
            privileges: {
                start: true,
                stop: true,
                restart: true,
                update: true
            }
        },
        data: function () {
            return {
                command: '',
                taskDate: '',
                taskRepeatInput: 1,
                taskRepeatRadio: 0,
                taskRepeatPeriod: 0,
                taskRepeatUnit: 'hours',

                selectedTaskIndex: null,
                errors: {},

                modalTitle: '',
                buttonName: this.trans('main.create'),
            }
        },
        methods: {
            createTask() {
                this.command = null;
                this.taskDate = null;
                this.taskRepeatInput = 1;
                this.taskRepeatPeriod = 1;
                this.taskRepeatUnit = 'hours';

                this.selectedTaskIndex = null;

                this.buttonName = this.trans('main.create');
                this.modalTitle = this.trans('servers_tasks.new_task');

                this.showModal();
            },
            editTask(index) {
                this.command = this.tasks[index].command;
                this.taskDate = this.tasks[index].execute_date;
                this.taskRepeatInput = '';
                this.repeat = this.tasks[index].repeat;

                const repeat = this.tasks[index].repeat_period.split(' ');

                this.taskRepeatPeriod = repeat[0];
                this.taskRepeatUnit = this.repeatUnitPlural(repeat[1]);

                this.selectedTaskIndex = index;

                this.buttonName = this.trans('main.save');
                this.modalTitle = this.trans('servers_tasks.edit_task');

                this.showModal();
            },
            sendTaskForm() {
                if (!this.checkForm()) {
                    return;
                }

                const form = {
                    server_id: this.serverId,
                    command: this.command,
                    execute_date: this.taskDate,
                    repeat: this.repeat,
                };

                form.repeat_period = form.repeat !== 1
                    ? this.taskRepeatPeriod + ' ' + this.taskRepeatUnit
                    : '';

                if (this.selectedTaskIndex === null) {
                    this.$store.dispatch('servers/storeTask', form)
                        .then(() => {
                            this.hideModal();
                        }).catch((e) => {
                            this.hideModal();

                            _.has(e, 'response.data.message')
                                ? gameap.alert(e.response.data.message)
                                : gameap.alert(e);
                        });
                } else {
                    this.$store.dispatch('servers/updateTask', {
                            taskIndex: this.selectedTaskIndex,
                            task: form
                        }).then(() => {
                            this.hideModal();
                        }).catch((e) => {
                            this.hideModal();

                            _.has(e, 'response.data.message')
                                ? gameap.alert(e.response.data.message)
                                : gameap.alert(e);
                        });
                }
            },
            checkForm() {
                this.resetErrors();
                let error = false;

                if (!this.command) {
                    error = true;
                    this.errors.task = this.trans('servers_tasks.errors.empty_task_command');
                }

                if (!this.taskDate) {
                    error = true;
                    this.errors.taskDate = this.trans('servers_tasks.errors.empty_task_date');
                }

                if (this.taskRepeatRadio === RADIO_REPEAT_CUSTOM
                    && (isNaN(parseInt(this.taskRepeatInput))
                        || this.taskRepeatInput < 1
                        || this.taskRepeatInput > 255)
                ) {
                    error = true;
                    this.errors.taskRepeatInput = this.trans('servers_tasks.errors.invalid_repeat_value');
                }

                if (this.repeat !== REPEAT_ONCE) {
                    if (!this.taskRepeatUnit) {
                        error = true;
                        this.errors.taskRepeatPeriod = this.trans('servers_tasks.errors.empty_period_unit');
                    } else if (!this.taskRepeatPeriod) {
                        error = true;
                        this.errors.taskRepeatPeriod = this.trans('servers_tasks.errors.empty_period');
                    } else if (this.taskRepeatUnit === 'minutes' && this.taskRepeatPeriod < 10) {
                        error = true;
                        this.errors.taskRepeatPeriod = this.trans('servers_tasks.errors.minimum_period');
                    }
                }

                return !error;
            },
            resetErrors() {
                this.errors = {
                    command: null,
                    taskDate: null,
                    taskRepeatRadio: null,
                    taskRepeatInput: null,
                    taskRepeatPeriod: null,
                };
            },
            formChange() {
                this.resetErrors();
            },
            deleteTask(taskIndex) {
                gameap.confirm(this.trans('servers_tasks.confirm_remove'), () => {
                    this.$store.dispatch('servers/destroyTask', taskIndex);
                })
            },
            humanRepeatText(repeatInt) {
                if (repeatInt === REPEAT_ENDLESSLY) {
                    return this.trans('servers_tasks.endlessly');
                }

                if (repeatInt === REPEAT_ONCE) {
                    return this.trans('servers_tasks.once');
                }

                return repeatInt;
            },
            repeatUnitPlural(unit) {
                switch (unit) {
                    case 'm':
                    case 'min':
                    case 'minute':
                        return 'minutes';

                    case 'h':
                    case 'hour':
                        return 'hours';

                    case 'd':
                    case 'day':
                        return 'days';

                    case 'w':
                    case 'week':
                        return 'weeks';

                    case 'month':
                        return 'months';

                    case 'y':
                    case 'year':
                        return 'years';
                }

                return unit;
            },
            showModal() {
                this.resetErrors();
                $('#createTaskModal').modal('show');
            },
            hideModal() {
                this.resetErrors();
                $('#createTaskModal').modal('hide');
            },
        },
        computed: {
            ...mapState({
                tasks: state => state.servers.tasks,
            }),
            repeat: {
                get() {
                    return parseInt(this.taskRepeatRadio === RADIO_REPEAT_CUSTOM
                        ? this.taskRepeatInput
                        : this.taskRepeatRadio);
                },
                set(repeat) {
                    repeat = parseInt(repeat);
                    if (repeat <= 0) {
                        this.taskRepeatRadio = 0;
                    } else if (repeat === REPEAT_ONCE) {
                        this.taskRepeatRadio = 1;
                    } else {
                        this.taskRepeatRadio = RADIO_REPEAT_CUSTOM;
                        this.taskRepeatInput = repeat;
                    }
                }
            },
            repeatMin: {
                get() {
                    if (this.taskRepeatUnit === 'minutes') {
                        return 10;
                    } else {
                        return 1;
                    }
                }
            },
            unitOptions: {
                get() {
                    return {
                        'minutes': this.pluralize('minute', parseInt(this.taskRepeatPeriod)),
                        'hours': this.pluralize('hour', parseInt(this.taskRepeatPeriod)),
                        'days': this.pluralize('day', parseInt(this.taskRepeatPeriod)),
                        'weeks': this.pluralize('week', parseInt(this.taskRepeatPeriod)),
                        'months': this.pluralize('month', parseInt(this.taskRepeatPeriod)),
                        'years': this.pluralize('year', parseInt(this.taskRepeatPeriod)),
                    };
                }
            }
        },
        mounted() {
            this.$store.dispatch('servers/setServerId', this.serverId);
            this.$store.dispatch('servers/fetchTasks');
        }
    }
</script>
