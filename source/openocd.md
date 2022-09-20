# Openocd

Openocd usually runs on a computer and control one or more chips on target
board via a debugger adapter. The adapters are usually connected to the
computer via USB, Ethernet or wireless physical interfaces. And different
protocols may go through them. On the target side, adapters genereate
JTAG/SWD/SWIM commands and receive responses. In short, debugger adapter
extends the computer's ability to communicate with target chip through
JTAG/SWD/SWIM transports. Above the transport layer, different chips implement
different high level protocols to expose internal debug resources.

Openocd utilizes libusb to communicate with debuggers through usb interface.
It's architecture agnostic and can be easily ported to ARM based computers.

## Command

Openocd utilizes libjim (a light weight tcl interpreter) to support an
extensible command interface and convenient scripting tool.

See jim.h for libjim API.

Following `command` is a derivation from `Jim_Cmd`. `jim_cmd->u.native.privData`
points back to `command`. All openocd commands are registered as `Jim_Cmd`
whose `jim_cmd->u.native.cmdProc` are `jim_command_dispatch`. The `privData`
identifies them. Jim supports multiple levels of subcommands and command
autocompletion.
```c
struct command {
    char *name;
    command_handler_t handler;

    Jim_CmdProc *jim_handler;
    void *jim_handler_data;

    /* Used only for target of target-prefixed cmd */
    struct target *jim_override_target;

    enum command_mode mode;
};

typedef __COMMAND_HANDLER((*command_handler_t));

#define __COMMAND_HANDLER(name, extra ...) \
        int name(struct command_invocation *cmd, ## extra)
```

- `mode`
    - `COMMAND_EXEC`
        Could only be executed after `init`.
        Comparison is done against command_context->mode

    - `COMMAND_CONFIG`
        Could only be executed before `init`.

    - `COMMAND_ANY`
        Could run in any context.

- `handler`, `jim_handler`
    Invoke `jim_handler` if available. Otherwise invoker handler with
    argv[0] shifted away.

- `jim_override_target`
    If set, `cmd_ctx->current_target_override` will be changed to this when
    executing the command to override `cmd_ctx->current_target`.


To describe what commands are available in different stages, context are divided
into config phase and execution phase, where different set of commands are
available. Some commands are stateless and could run in both stages.


Beside stage info, command context also contains an tcl engine `Jim_Interp`,
and target device descriptor. Output handler is also per context to redirect 
command output.

```c
struct command_context {
    Jim_Interp *interp;

    enum command_mode mode;		/* current mode */

    /* The target set by 'targets xx' command or the latest created */
    /* If set overrides current_target
     * It happens during processing of
     *  1) a target prefixed command
     *  2) an event handler
     * Pay attention to reentrancy when setting override.
     */
    struct target *current_target;
    struct target *current_target_override;

    command_output_handler_t output_handler;
    void *output_handler_priv;

    /* to link all `help_entry`, help and usage info of registered commands */
    struct list_head *help_list;
};

typedef int (*command_output_handler_t)(struct command_context *context, const char *line);
```

```c
struct help_entry {
    struct list_head lh;
    char *cmd_name;
    char *help;
    char *usage;
};
```

Commands should be registered by filling in one or more of these structures
and passing them to `register_commands`. To support multiple levels of subcmds,
command registration could embed an extra array of `command_registration`.
The "chain" forms a subcmd tree hierarchy.
```c
struct command_registration {
    const char *name;               /* unique in a context */
    command_handler_t handler;
    Jim_CmdProc *jim_handler;
    enum command_mode mode; /* The command mode(s) in which this command may be run. */

    /* help and usage info strings */
    const char *help;
    const char *usage;

    /**
     * If non-NULL, the commands in chain will be registered in
     * the same context and scope of this registration record.
     * This allows modules to inherit lists commands from other
     * modules.
     */
    const struct command_registration *chain;
};
```


`register_commands` series register openocd customized commands to Jim core.

`data` is new command's `jim_handler_data`.
`override_target` is `jim_override_target`
```c
int __register_commands(struct command_context *cmd_ctx, const char *cmd_prefix,
    const struct command_registration *cmds, void *data,
    struct target *override_target)
{
    int retval = ERROR_OK;
    unsigned i;
    for (i = 0; cmds[i].name || cmds[i].chain; i++) {
        const struct command_registration *cr = cmds + i;

        struct command *c = NULL;
        if (NULL != cr->name) {
            c = register_command(cmd_ctx, cmd_prefix, cr);
            if (NULL == c) {
                retval = ERROR_FAIL;
                break;
            }
            c->jim_handler_data = data;
            c->jim_override_target = override_target;
        }
        if (NULL != cr->chain) {
            if (cr->name) {
                if (cmd_prefix) {
                    char *new_prefix = alloc_printf("%s %s", cmd_prefix, cr->name);
                    if (!new_prefix) {
                        retval = ERROR_FAIL;
                        break;
                    }
                    retval = __register_commands(cmd_ctx, new_prefix, cr->chain, data, override_target);
                    free(new_prefix);
                } else {
                    retval = __register_commands(cmd_ctx, cr->name, cr->chain, data, override_target);
                }
            } else {
                retval = __register_commands(cmd_ctx, cmd_prefix, cr->chain, data, override_target);
            }
            if (ERROR_OK != retval)
                break;
        }
    }
    if (ERROR_OK != retval) {
        for (unsigned j = 0; j < i; j++)
            unregister_command(cmd_ctx, cmd_prefix, cmds[j].name);
    }
    return retval;
}
```

```c
static struct command *register_command(struct command_context *context,
    const char *cmd_prefix, const struct command_registration *cr)
{
    char *full_name;

    if (!context || !cr->name)
        return NULL;

    if (cmd_prefix)
        full_name = alloc_printf("%s %s", cmd_prefix, cr->name);
    else
        full_name = strdup(cr->name);
    if (!full_name)
        return NULL;

    /* Check if there is already a registered Jim_Cmd with the "full_name" */
    struct command *c = command_find_from_name(context->interp, full_name);
    if (c) {
        /* TODO: originally we treated attempting to register a cmd twice as an error
         * Sometimes we need this behaviour, such as with flash banks.
         * http://www.mail-archive.com/openocd-development@lists.berlios.de/msg11152.html */
        LOG_DEBUG("command '%s' is already registered", full_name);
        free(full_name);
        return c;
    }

    c = command_new(context, full_name, cr);
    if (!c) {
        free(full_name);
        return NULL;
    }

    if (false) /* too noisy with debug_level 3 */
        LOG_DEBUG("registering '%s'...", full_name);
    int retval = Jim_CreateCommand(context->interp, full_name,
                jim_command_dispatch, c, command_free);
    if (retval != JIM_OK) {
        command_run_linef(context, "del_help_text {%s}", full_name);
        command_run_linef(context, "del_usage_text {%s}", full_name);
        free(c);
        free(full_name);
        return NULL;
    }

    free(full_name);
    return c;
}
```

Command Handler/Dispatcher:
```c
static int jim_command_dispatch(Jim_Interp *interp, int argc, Jim_Obj * const *argv)
{
    script_debug(interp, argc, argv);

    /* check subcommands */
    if (argc > 1) {
        char *s = alloc_printf("%s %s", Jim_GetString(argv[0], NULL), Jim_GetString(argv[1], NULL));
        Jim_Obj *js = Jim_NewStringObj(interp, s, -1);
        Jim_IncrRefCount(js);
        free(s);
        Jim_Cmd *cmd = Jim_GetCommand(interp, js, JIM_NONE);
        if (cmd) {
            int retval = Jim_EvalObjPrefix(interp, js, argc - 2, argv + 2);
            Jim_DecrRefCount(interp, js);
            return retval;
        }
        Jim_DecrRefCount(interp, js);
    }

    struct command *c = jim_to_command(interp);
    if (!c->jim_handler && !c->handler) {
        Jim_EvalObjPrefix(interp, Jim_NewStringObj(interp, "usage", -1), 1, argv);
        return JIM_ERR;
    }

    struct command_context *cmd_ctx = current_command_context(interp);

    if (!command_can_run(cmd_ctx, c, Jim_GetString(argv[0], NULL)))
        return JIM_ERR;

    target_call_timer_callbacks_now();

    /*
     * Black magic of overridden current target:
     * If the command we are going to handle has a target prefix,
     * override the current target temporarily for the time
     * of processing the command.
     * current_target_override is used also for event handlers
     * therefore we prevent touching it if command has no prefix.
     * Previous override is saved and restored back to ensure
     * correct work when jim_command_dispatch() is re-entered.
     */
    struct target *saved_target_override = cmd_ctx->current_target_override;
    if (c->jim_override_target)
        cmd_ctx->current_target_override = c->jim_override_target;

    int retval = exec_command(interp, cmd_ctx, c, argc, argv);

    if (c->jim_override_target)
        cmd_ctx->current_target_override = saved_target_override;

    return retval;
}
```

Run `c->jim_handler` if it's registered.  Or run `c->handler`
```c
static int exec_command(Jim_Interp *interp, struct command_context *cmd_ctx,
        struct command *c, int argc, Jim_Obj * const *argv)
{
    if (c->jim_handler)
        return c->jim_handler(interp, argc, argv);

    /* use c->handler */
    unsigned int nwords;
    char **words = script_command_args_alloc(argc, argv, &nwords);
    if (!words)
        return JIM_ERR;

    int retval = run_command(cmd_ctx, c, (const char **)words, nwords);
    script_command_args_free(words, nwords);
    return command_retval_set(interp, retval);
}
```

```c
static int run_command(struct command_context *context,
    struct command *c, const char **words, unsigned num_words)
{
    struct command_invocation cmd = {
        .ctx = context,
        .current = c,
        .name = c->name,
        .argc = num_words - 1,
        .argv = words + 1,
    };

    cmd.output = Jim_NewEmptyStringObj(context->interp);
    Jim_IncrRefCount(cmd.output);

    int retval = c->handler(&cmd);
    if (retval == ERROR_COMMAND_SYNTAX_ERROR) {
        /* Print help for command */
        command_run_linef(context, "usage %s", words[0]);
    } else if (retval == ERROR_COMMAND_CLOSE_CONNECTION) {
        /* just fall through for a shutdown request */
    } else {
        if (retval != ERROR_OK)
            LOG_DEBUG("Command '%s' failed with error code %d",
                        words[0], retval);
        /* Use the command output as the Tcl result */
        Jim_SetResult(context->interp, cmd.output);
    }
    Jim_DecrRefCount(context->interp, cmd.output);

    return retval;
}
```


## Adapter/Interface

`adapter_driver` represents a driver talking to debug adapter, like jlink,
cmsis-dap, ulink, bus blaster etc. Every adapter shall declare supported
transports and provide methods to transfer high level commands over these
transports.

```c
struct adapter_driver {
    const char * const name;
    const char * const *transports; /* transports supported (NULL terminated) */

    const struct command_registration *commands;    /* adapter cmds */

    int (*init)(void);
    int (*quit)(void);
    int (*reset)(int srst, int trst);

    int (*speed)(int speed);    /* set speed */
    int (*khz)(int khz, int *jtag_speed);
    int (*speed_div)(int speed, int *khz);

    int (*power_dropout)(int *power_dropout);

    int (*srst_asserted)(int *srst_asserted);

    int (*config_trace)(bool enabled, enum tpiu_pin_protocol pin_protocol,
        uint32_t port_size, unsigned int *trace_freq,
        unsigned int traceclkin_freq, uint16_t *prescaler);

    int (*poll_trace)(uint8_t *buf, size_t *size);

    struct jtag_interface *jtag_ops;
    const struct swd_driver *swd_ops;
    const struct dap_ops *dap_jtag_ops;
    const struct dap_ops *dap_swd_ops;
    const struct swim_driver *swim_ops;
};
```
Details:

- `commands`
    To expose additional features not covered by the standard command set.

- `init`, `quit`
    `init` is invoked by `adapter_init` to check existence of the adapter and
	prepare resources.
    
- `reset`
    Control (assert/deassert) the signals SRST and TRST on the interface.
    This function is synchronous and should be called after the adapter
    queue has been properly flushed.
    This function is optional.
    Adapters that don't support resets can either not define this function
    or return an error code.
    Adapters that don't support one of the two reset should ignore the
    request to assert the missing signal and eventually log an error.
    
    @param srst 1 to assert SRST, 0 to deassert SRST.
    @param trst 1 to assert TRST, 0 to deassert TRST.

- `speed`, `khz`, `speed_div`
    Adapter speed control. User could only see clock freq in khz.

    `speed` Sets the interface speed.
    `khz` transforms khz to adapter specific speed
    `speed_div` transforms speed to khz

- `power_dropout`
    Read and clear the power dropout flag. Note that a power dropout
    can be transitionary, easily much less than a ms.
    
    To find out if the power is *currently* on, one must invoke this
    method twice.  Once to clear the power dropout flag and a second
    time to read the current state.  The default implementation
    never reports power dropouts.

- `srst_asserted`
    Read and clear the srst asserted detection flag.
    
    Like power_dropout this does *not* read the current
    state.  SRST assertion is transitionary and may be much
    less than 1ms, so the interface driver must watch for these
    events until this routine is called.
    
    param srst_asserted On return, indicates whether SRST has
    been asserted.
    
- `config_trace`
    Configure trace parameters for the adapter
    
    @param enabled Whether to enable trace
    @param pin_protocol Configured pin protocol
    @param port_size Trace port width for sync mode
    @param trace_freq A pointer to the configured trace
    frequency; if it points to 0, the adapter driver must write
    its maximum supported rate there
    @param traceclkin_freq TRACECLKIN frequency provided to the TPIU in Hz
    @param prescaler Pointer to the SWO prescaler calculated by the
    adapter
    @returns ERROR_OK on success, an error code on failure.

- `poll_trace`
    Poll for new trace data

    @param buf A pointer to buffer to store received data
    @param size A pointer to buffer size; must be filled with
    the actual amount of bytes written
    
    @returns ERROR_OK on success, an error code on failure.
    
- `jtag_ops`
    jtag method used when jtag transport is selected.

    - `execute_queue`
        Issue queued cmds to adapter. Adapter executes them.
		Return after handling all command responses.

- `swd_ops`
    swd methods used when swd transport is selected.

	- `init`
		Initialize the debug link so it can perform SWD operations.

		As an example, this would switch a dual-mode debug adapter
		into SWD mode and out of JTAG mode.

	- `switch_seq`
		Queue a special SWDIO sequence (This is adi v5 specific). Supported seq:
		
		`LINE_RESET`
		`JTAG_TO_SWD`
		`JTAG_TO_DORMANT`
		`SWD_TO_JTAG`
		`SWD_TO_DORMANT`
		`DORMANT_TO_SWD`
		
	- `read_reg`, `write_reg`, `run`
		Queue read/write of an AP or DP register.
		
		Command byte is encoded with APnDP/RnW/addr/parity bits
		`ap_delay_hint` Number of idle cycles that may be
		needed after an AP access to avoid WAITs
		
		`run` Executes any queued transactions and collect the result.
		
	- `trace`
		Configures data collection from the Single-wire trace (SWO) signal.
		@param swo true if SWO data collection should be routed.
		
		For example,  some debug adapters include a UART which
		is normally connected to a microcontroller's UART TX,
		but which may instead be connected to SWO for use in
		collecting ITM (and possibly ETM) trace data.
		
		@return ERROR_OK on success, else a negative fault code.
 
- `dap_jtag_ops`
- `dap_swd_ops`
- `swim_ops`


All supported adapters are defined in static array `adapter_drivers`. Enable
corresponding options when running `configure` script to add support for your
debug adapter.
At run time, User should select the adapter in use in config script using
`adapter driver <interface>`.

```c
struct adapter_driver *adapter_drivers[] = {
        ...
#if BUILD_JLINK == 1
        &jlink_adapter_driver,
#endif
        ...
#if BUILD_CMSIS_DAP_USB == 1 || BUILD_CMSIS_DAP_HID == 1
        &cmsis_dap_adapter_driver,
#endif
        ...
        NULL,
    };

struct adapter_driver *adapter_driver;
```

For each openocd instance, only one adapter can be used. `adapter_config`
tracks global configuration for it.
```c
static struct {
    bool adapter_initialized;
    char *usb_location;
    char *serial;
    enum adapter_clk_mode clock_mode;
    int speed_khz;
    int rclk_fallback_speed_khz;
	struct adapter_gpio_config gpios[ADAPTER_GPIO_IDX_NUM];
	bool gpios_initialized; /* Initialization of GPIOs to their unset values performed at run time */
} adapter_config;
```
- `serial`, `usb_location`
	Serial number is used to identify an adapter if multiple adapters are
	available on the computer.

- `clock_mode`
	CLOCK_MODE_KHZ
		Fixed transport speed in given baudrate.
	CLOCK_MODE_RCLK
		Use adaptive clocking if possible, else to use the fallback speed.

```c
```

Common adapter commands:
- `adapter`
	- `driver`
		Select adapter driver, and get supported transports.
	- `speed`
		Set or Get speed of transport side?.
	- `serial`
		Choose target adapter serial number
	- `list`
		List all supported adapters.
	- `name`
		Return current selected adapter.
	- `srst`
		- `delay`
			Set/Get nsrst delay(openocd sleep) in ms after release
		- `pulse_width`
			Set/Get nsrst delay(openocd sleep) in ms after assert
	- `transports`
		Allow all compiled transports TODO
	- `usb`
		- `location`
			Display or set the location of the USB adapter
	- `assert|deassert [srst|trst [assert|deassert srst|trst]]`
		Display or set trst and srst lines
	- `gpio`

- `reset_config [none|trst_only|srst_only|trst_and_srst]
	[srst_pulls_trst|trst_pulls_srst|combined|separate]
	[srst_gates_jtag|srst_nogate] 
	[trst_push_pull|trst_open_drain] 
	[srst_push_pull|srst_open_drain] 
	[connect_deassert_srst|connect_assert_srst]`

There are two kinds of reset: system reset and tap reset. System reset pins
out from jtag/swd transport to target board. tap reset is defined
only in jtag protocol. Target may support either of them, both or none.

`reset_config` tells openocd trst and srst configs of target board.

`srst_gates_jtag`/`srst_nogate` tells if system reset affect target jtag
functionality. If `srst_nogate`, `connect_assert_srst` tells that srst should
be asserted when connecting jtag (used if jtag pins are programmed to other
functions very early or suspended by software).

Both srst and trst can be push-pull or open-drain?

`adapter assert|deassert [srst|trst [assert|deassert srst|trst]]` control srst
and trst lines. Only jtag support trst. trst can be signaled through separate
pin or jtag reset command. `driver->reset` is called to do resets.

##### cmsis daplink

```c
struct cmsis_dap {
    struct cmsis_dap_backend_data *bdata;
    const struct cmsis_dap_backend *backend;
    uint16_t packet_size;		/* from endpoint.wMaxPacketSize */
    int packet_count;
    uint8_t *packet_buffer;
    uint16_t packet_buffer_size;
    uint8_t *command;	/* point to packet_buffer, used to fill cmd */
    uint8_t *response;	/* point to packet buffer, used to refer resp data */
    uint16_t caps;
    uint8_t mode;
    uint32_t swo_buf_sz;
    bool trace_enabled;
};

static struct cmsis_dap *cmsis_dap_handle;
```

Legacy CMSIS-DAP is based on USB HID. We only focus on the advanced, bulk
transfer based  USB Vendor Specific Class implementation.

"adapter init" invokes `backend->open` to determine following data of bulk
based CMSIS-DAP.
```c
struct cmsis_dap_backend_data {
    struct libusb_context *usb_ctx;
    struct libusb_device_handle *dev_handle;
    unsigned int ep_out;
    unsigned int ep_in;
    int interface;
};
```

```c
struct cmsis_dap_backend {
    const char *name;	/* usb_bulk or hid */
    int (*open)(struct cmsis_dap *dap, uint16_t vids[], uint16_t pids[], const char *serial);
    void (*close)(struct cmsis_dap *dap);
    int (*read)(struct cmsis_dap *dap, int timeout_ms);
    int (*write)(struct cmsis_dap *dap, int len, int timeout_ms);
    int (*packet_buffer_alloc)(struct cmsis_dap *dap, unsigned int pkt_sz);
};
```
- `read`
	No way to return bytes read.
	Read data to `dap->packet_buffer`. Trailing contents are cleared to zeros.
- `write`
	Write data of given length in packet_buffer.

CMSIS-DAP wraps a transfer interface which sends out the cmd in `packet_buffer`
and reads back a response in the `packet_buffer` too. Caller should use
`dap->command` to fill command but reference it using `dap->response` when
parsing response.

The first byte in command buffer determines the command type, followed by
arguments. The first byte of response shall be the echo of the command.
Supported commands:

- `CMD_DAP_INFO`
	Get daplink info. second byte is the info type listed below.
	- `INFO_ID_VENDOR`
	- `INFO_ID_PRODUCT`
	- `INFO_ID_SERNUM`
	- `INFO_ID_FW_VER`
	- `INFO_ID_TD_VEND`
	- `INFO_ID_TD_NAME`
	- `INFO_ID_CAPS`
		- `INFO_CAPS_SWD`
		- `INFO_CAPS_JTAG`
		- `INFO_CAPS_SWO_UART`
		- `INFO_CAPS_SWO_MANCHESTER`
		- `INFO_CAPS_ATOMIC_CMDS`
		- `INFO_CAPS_TEST_DOMAIN_TIMER`
		- `INFO_CAPS_SWO_STREAMING_TRACE`
		- `INFO_CAPS_UART_PORT`
		- `INFO_CAPS_USB_COM_PORT`
		- `INFO_CAPS__NUM_CAPS`
	- `INFO_ID_PKT_CNT`
	- `INFO_ID_PKT_SZ`
	- `INFO_ID_SWO_BUF_SZ`
- `CMD_DAP_LED`
	Control following leds. byte 1 is led id. byte 2 is state, 0-off, 1-on.
	- `LED_ID_CONNECT`
	- `LED_ID_RUN`

- `CMD_DAP_CONNECT`
	- `CONNECT_DEFAULT`
	- `CONNECT_SWD`
	- `CONNECT_JTAG`
- `CMD_DAP_DISCONNECT`

- `CMD_DAP_TFER_CONFIGURE`
- `CMD_DAP_TFER`
- `CMD_DAP_TFER_BLOCK`
- `CMD_DAP_TFER_ABORT`


- `CMD_DAP_WRITE_ABORT`
- `CMD_DAP_DELAY`
- `CMD_DAP_RESET_TARGET`

- `CMD_DAP_DELAY`
- `CMD_DAP_SWJ_PINS`
	- `SWJ_PIN_TCK`
	- `SWJ_PIN_TMS`
	- `SWJ_PIN_TDI`
	- `SWJ_PIN_TDO`
	- `SWJ_PIN_TRST`
	- `SWJ_PIN_SRST`
- `CMD_DAP_SWJ_CLOCK`
- `CMD_DAP_SWJ_SEQ`
	clock a sequence of bits out on TMS, to change JTAG states.

- `CMD_DAP_SWD_CONFIGURE`
- `CMD_DAP_SWD_SEQUENCE`

- `CMD_DAP_JTAG_SEQ`
- `CMD_DAP_JTAG_CONFIGURE`
- `CMD_DAP_JTAG_IDCODE`

- swo cmds TODO

```c
```

```c
struct adapter_driver cmsis_dap_adapter_driver = {
    .name = "cmsis-dap",
    .transports = cmsis_dap_transport,  /* swd and jtag */
    .commands = cmsis_dap_command_handlers,

    .init = cmsis_dap_init,
    .quit = cmsis_dap_quit,
    .reset = cmsis_dap_reset,

    /* cmsis-dap use khz as speed */
    .speed = cmsis_dap_speed,
    .khz = cmsis_dap_khz,
    .speed_div = cmsis_dap_speed_div,

    .jtag_ops = &cmsis_dap_interface,
    .swd_ops = &cmsis_dap_swd_driver,
};
```

- `cmsis_dap_init`
	- `cmsis_dap_open`
		Create a `cmsis_dap`.
		Open selected backend by `backend->open`, vid,pid, and serial are used
		if set to locate the adapter.
		This callback detects target usb device. setup backend data, and init
		package buffer.
	- `cmsis_dap_flush_read`
		Read stale data out.
	- `cmsis_dap_get_caps_info`
	- `cmsis_dap_get_version_info`
	- `cmsis_dap_get_serial_info`
	- `cmsis_dap_cmd_dap_connect(CONNECT_JTAG)`
	- `cmsis_dap_cmd_dap_info(INFO_ID_PKT_SZ, &data)`
	- `cmsis_dap_cmd_dap_info(INFO_ID_PKT_CNT, &data)`
	- `cmsis_dap_get_status`
	- `cmsis_dap_cmd_dap_swj_clock(adapter_get_speed_khz())`
	...


`cmsis_dap_command_handlers`:
- `cmsis-dap`
    - `info`
        log out FW version and pin states.
    - `cmd`
        Transfer raw commands and print response.
- `cmsis_dap_vid_pid`
    Set(Select) `cmsis_dap_vid` and `cmsis_dap_pid`
- `cmsis_dap_backend`
    Set the communication backend to use (USB bulk or HID). Affect `cmsis_dap_backend`
- `cmsis_dap_usb` (usb bulk backend-specific commands)
    - `interface`
        Set the USB interface number to use (Bulk only) to distinguish from
		other bulk interfaces. Affect `cmsis_dap_usb_interface`
		Used in config phase.
   

##### jlink


```c
```


```c
```

## Transport

OpenOCD instructs debug adapter to talk targets through some kind of protocol
over physical link that probably has target-specific aspects.

A "transport" reflects electrical protocol to the target, e.g. jtag, swd, spi,
uart, ... NOT the messaging protocols layered over it (e.g. JTAG may carry
eICE, CoreSight, Nexus, OnCE, and more protocols).

struct `transport` is a Wrapper for transport lifecycle operations.

In addition to the lifecycle operations in this structure, a transport
also involves an interface supported by debug adapters and used by components
such as debug targets. For non-debug transports, there may be interfaces used to
write to flash chips.
```c
struct transport {
    const char *name;       /* "jtag", * "swd", "AVR_ISP" and more. */

    int (*select)(struct command_context *ctx);

    int (*init)(struct command_context *ctx);

    int (*override_target)(const char **targetname);

    struct transport *next;     /* linked by "transport_list" after registered */
};

/* transports supported by current adapter*/
static const char * const *allowed_transports;

static struct transport *transport_list;

static struct transport *session;	/* current transport */
```
- `select`
    When a transport is selected, this method registers its commands and
	activates the transport (e.g. resets the link).
    
    After those commands are registered, they will often
    be used for further configuration of the debug link.
    
- `init`
    server startup uses this method to validate transport configuration.
	(For example, with JTAG this interrogates the scan chain against the
	 list of expected TAPs.)

- `override_target`
    Optional. If defined, allows transport to override target
    name prior to initialisation.
    

when registering a debug adapter, transports it supports are set allowed.
One debug adapter may support multiple transports. User should use
`transport select <transport>` to select the actually used transport.

transport commands:
- `transport`
    - `init`
        Call `session->init`. `transport select <transport>` must have been called.
    - `list`
        Print all registered transports (`transport_list`).
    - `select`
        Select one from `allowed_transports` to be `session` and invoke its
        `select` method.
		This callback usually registers transport specific commands, and may
		also reset the link, e.g. forcing it to JTAG or SWD mode.


Transports like jtag and swd are registered at construction time, and usually
be selected by board/target cfg files.

```c
static struct transport jtag_transport = {
    .name = "jtag",
    .select = jtag_select,
    .init = jtag_init,
};
```

```c
static int jtag_select(struct command_context *ctx)
{
    int retval;

    /* NOTE:  interface init must already have been done.
     * That works with only C code ... no Tcl glue required.
     */

    retval = jtag_register_commands(ctx);

    if (retval != ERROR_OK)
        return retval;

    retval = svf_register_commands(ctx);

    if (retval != ERROR_OK)
        return retval;

    retval = xsvf_register_commands(ctx);

    if (retval != ERROR_OK)
        return retval;

    return ipdbg_register_commands(ctx);
}
```

The registered jtag transport commands - `jtag_command_handlers`:
- `jtag_flush_queue_sleep`
    Set the manual delay in ms after executing a jtag command queue. For debug
    purpose.
- `jtag_rclk`
    Get/Set jtag speed. (invoke `adapter->khz` and `adapter->speed`)
- `jtag_ntrst_delay`
    Get/Set `jtag_ntrst_delay`. Minimal delay between deasserting ntrst and
	executing next command. trst delay is set by "reset_config".
- `jtag_ntrst_assert_width`
    Get/Set `jtag_ntrst_assert_width`. Time after asserting ntrst

- `scan_chain`
	Print current scan chain config
- `runtest`
    Run jtag test from idle to idle state in given cycles.
- `irscan`
    Write ir to a tap, and optional move to given stable state.
    "[tap_name instruction]* ['-endstate' state_name]"
- `verify_ircapture`
    Get/Set `jtag_verify_capture_ir` flag, which determines whether ir verification
    will be done (shift ir out and check).
- `verify_jtag`
    Get/Set `jtag_verify`, 
- `tms_sequence`
    Set "short/long" to use different types of tms sequences.
- `wait_srst_deassert`
    Used after issuing srst cmd, wait srst to be deasserted.
- `jtag` perform jtag tap actions
    - `init`
        Call `jtag_init` (only once), which invokes `adapter_init`, and reset
        target, and invoke `jtag_init` proc.
    - `arp_init`
    - `arp_init-reset`
    - `newtap <chip> <tap> -irlen <count> [-enable|-disable] [-expected_id <number>] [-ignore-version] [-ircapture <number>] [-mask <number>]`
        Create a jtag_tap
    - `tapisenabled`
    - `tapenable`
    - `configure`
    - `cget`
    - `names`
    - `drscan`
    - `flush_count`
    - `path_move`

`jtag_tap` represents tap/dap created by both `jtag newtap` and swd.
```c
struct jtag_tap {
    char *chip;
    char *tapname;
    char *dotted_name;  /* <chip>.<tapname> */

    int abs_chain_position;		/* index of the tap in the jtag chain */

    bool disabled_after_reset;
    bool enabled;

    int ir_length; /**< size of instruction register, set by -irlen */
    uint32_t ir_capture_value;	/* set by -ircapture */
    uint8_t *expected; /**< Capture-IR expected value */
    uint32_t ir_capture_mask;	/* set by -irmask */
    uint8_t *expected_mask; /**< Capture-IR expected mask */

    uint32_t idcode; /**< device identification code */
    bool hasidcode;

    /* Array of expected identification codes, set by one or more -expected_id */
    uint32_t *expected_ids;
    uint8_t expected_ids_cnt;

    /** Flag saying whether to ignore version field in expected_ids[] */
    bool ignore_version;

    uint8_t *cur_instr;		/** current instruction */
    int bypass;		/** Bypass register selected */

    struct jtag_tap_event_action *event_action;

    struct jtag_tap *next_tap;  /* linked onto "__jtag_all_taps" */

    void *priv;
};
```


##### SWD Transport

```c
static struct transport swd_transport = {
    .name = "swd",
    .select = swd_select,
    .init = swd_init,
};
```
`swd_select` registers swd commands and invokes `adapter->swd_ops->init` to do
adapter swd init.

   
 
SWD commands:
- `swd`
    - `newdap`
        Create a swd tap.

```c
```

```c
```



##### JTAG Specific Operations


`jtag_ops` in `adapter_driver` Represents a driver for a jtag interface.

```c
struct jtag_interface {
    /* capabilities exposed by this driver.  */
    unsigned supported;
#define DEBUG_CAP_TMS_SEQ   (1 << 0)

    /* Execute queued commands. Queued "jtag_command" are gathered in link list
     * "jtag_command_queue"  */
    int (*execute_queue)(void);
};
```

Adapters supporting JTAG should be able to handle a common set of JTAG commands
listed below.
Driver will pass these commands through for example USB to adapter.
Adapter firmware will talk to target using JTAG transport.
```c
enum jtag_command_type {
    JTAG_SCAN         = 1,
    /* JTAG_TLR_RESET's non-minidriver implementation is a
     * vestige from a statemove cmd. The statemove command
     * is obsolete and replaced by pathmove.
     *
     * pathmove does not support reset as one of it's states,
     * hence the need for an explicit statemove command.
     */
    JTAG_TLR_RESET    = 2,
    JTAG_RUNTEST      = 3,
    JTAG_RESET        = 4,
    JTAG_PATHMOVE     = 6,
    JTAG_SLEEP        = 7,
    JTAG_STABLECLOCKS = 8,
    JTAG_TMS          = 9,
};

typedef enum tap_state {
    TAP_INVALID = -1,

    /* Proper ARM recommended numbers */
    TAP_DREXIT2 = 0x0,
    TAP_DREXIT1 = 0x1,
    TAP_DRSHIFT = 0x2,
    TAP_DRPAUSE = 0x3,
    TAP_IRSELECT = 0x4,
    TAP_DRUPDATE = 0x5,
    TAP_DRCAPTURE = 0x6,
    TAP_DRSELECT = 0x7,
    TAP_IREXIT2 = 0x8,
    TAP_IREXIT1 = 0x9,
    TAP_IRSHIFT = 0xa,
    TAP_IRPAUSE = 0xb,
    TAP_IDLE = 0xc,
    TAP_IRUPDATE = 0xd,
    TAP_IRCAPTURE = 0xe,
    TAP_RESET = 0x0f,
} tap_state_t;

struct jtag_command {
    union jtag_command_container cmd;
    enum jtag_command_type type;
    struct jtag_command *next;
};

union jtag_command_container {
    struct scan_command *scan;
    struct statemove_command *statemove;	/* move states one by one */
    struct pathmove_command *pathmove;
    struct runtest_command *runtest;
    struct stableclocks_command *stableclocks;
    struct reset_command *reset;
    struct end_state_command *end_state;
    struct sleep_command *sleep;
    struct tms_command *tms;
};

struct tms_command {
    unsigned num_bits;
    const uint8_t *bits;
};

struct scan_command {
    bool ir_scan;	/* ir/dr scan */

    int num_fields;		/* one field for a tap on the chain */
    struct scan_field *fields;

    tap_state_t end_state;
};

struct scan_field {
    int num_bits;	/* ir len or dr len */
    const uint8_t *out_value;
    uint8_t *in_value;

    uint8_t *check_value;
    uint8_t *check_mask;
};

extern struct jtag_command *jtag_command_queue;
```
`jtag_command_queue` queues commands for next execution. They are
consumed by `adapter->jtag_ops->execute_queue`.

Details of jtag commands:
- `JTAG_SCAN`
- `JTAG_TLR_RESET`   
- `JTAG_RUNTEST`     
- `JTAG_RESET`       
- `JTAG_PATHMOVE`    
- `JTAG_SLEEP`       
- `JTAG_STABLECLOCKS`
- `JTAG_TMS`
	Clock out a TMS sequence.


##### SWD Specific Operations

  
   

    
    



```c
```

## ARM DAP

ARM DAP is the debug port to one or more APs(access port). Each AP cuuld
access one target.
DAP is a special jtag tap.


`adiv5_dap` represents an ARM Debug Interface (v5) Debug Access Port (DAP).
A DAP has two types of component:  one Debug Port (DP), which is a
transport agent; and at least one Access Port (AP), controlling
resource access.

There are two basic DP transports: JTAG, and ARM's low pin-count SWD.
Accordingly, this interface is responsible for hiding the transport
differences so upper layer code can largely ignore them.

When the chip is implemented with JTAG-DP or SW-DP, the transport is
fixed as JTAG or SWD, respectively.  Chips incorporating SWJ-DP permit
a choice made at board design time (by only using the SWD pins), or
as part of setting up a debug session (if all the dual-role JTAG/SWD
signals are available).

```c
struct arm_dap_object {
    struct list_head lh;	/* `all_daps` links this */
    struct adiv5_dap dap;
    char *name;
    const struct swd_driver *swd;
};
```
```c
struct adiv5_dap {
    const struct dap_ops *ops;

    /* dap transaction list for WAIT support */
    struct list_head cmd_journal;

    /* pool for dap_cmd objects */
    struct list_head cmd_pool;

    /* number of dap_cmd objects in the pool */
    size_t cmd_pool_size;

    struct jtag_tap *tap;   /* the internal jtag tap */

    uint32_t dp_ctrl_stat; /* Control config */

    struct adiv5_ap ap[DP_APSEL_MAX + 1];

    /* The current manually selected AP by the "dap apsel" command */
    uint32_t apsel;

    /**
     * Cache for DP_SELECT register. A value of DP_SELECT_INVALID
     * indicates no cached value and forces rewrite of the register.
     */
    uint32_t select;

    /* information about current pending SWjDP-AHBAP transaction */
    uint8_t  ack;

    /**
     * Holds the pointer to the destination word for the last queued read,
     * for use with posted AP read sequence optimization.
     */
    uint32_t *last_read;

    /* The TI TMS470 and TMS570 series processors use a BE-32 memory ordering
     * despite lack of support in the ARMv7 architecture. Memory access through
     * the AHB-AP has strange byte ordering these processors, and we need to
     * swizzle appropriately. */
    bool ti_be_32_quirks;

    /**
     * STLINK adapter need to know if last AP operation was read or write, and
     * in case of write has to flush it with a dummy read from DP_RDBUFF
     */
    bool stlink_flush_ap_write;

    /**
     * Signals that an attempt to reestablish communication afresh
     * should be performed before the next access.
     */
    bool do_reconnect;

    /** Flag saying whether to ignore the syspwrupack flag in DAP. Some devices
     *  do not set this bit until later in the bringup sequence */
    bool ignore_syspwrupack;
};
```

```c
struct adiv5_ap {
    struct adiv5_dap *dap;		/* DAP this AP belongs to. */

    uint8_t ap_num;

    /* Default value for (MEM-AP) AP_REG_CSW register.  */
    uint32_t csw_default;

    /**
     * Cache for (MEM-AP) AP_REG_CSW register value.  This is written to
     * configure an access mode, such as autoincrementing AP_REG_TAR during
     * word access.  "-1" indicates no cached value.
     */
    uint32_t csw_value;

    /**
     * Cache for (MEM-AP) AP_REG_TAR register value This is written to
     * configure the address being read or written
     * "-1" indicates no cached value.
     */
    target_addr_t tar_value;

    /**
     * Configures how many extra tck clocks are added after starting a
     * MEM-AP access before we try to read its status (and/or result).
     */
    uint32_t memaccess_tck;

    /* Size of TAR autoincrement block, ARM ADI Specification requires at least 10 bits */
    uint32_t tar_autoincr_block;

    /* true if packed transfers are supported by the MEM-AP */
    bool packed_transfers;

    /* true if unaligned memory access is not supported by the MEM-AP */
    bool unaligned_access_bad;

    /* true if tar_value is in sync with TAR register */
    bool tar_valid;

    /* MEM AP configuration register indicating LPAE support */
    uint32_t cfg_reg;
};
```

arm `dap_commands`:

- `dap`
    - `create <name> -chain-position <jtag_tap> [<dap_options>]`
        create an ARM DAP attached to a jtag tap, and register dap instance
		commands.
    - `names`
        List names of all created daps on "all_dap".
    - `init`
    - `info`

dap instance command group:
- `<dap_name>`
    - `info`
	- `apsel`
	- `apcsw`
	- `apid`
	- `apreg`
	- `dpreg`
	- `baseaddr`
	- `memaccess`
	- `ti_be_32_quirks`

```c
```

## Target

A debuggable target (a processor?) on scan chain or arm dap.
```c
struct target {
    struct target_type *type;   /* target type definition (name, access functions) */
    char *cmd_name;             /* tcl Name of target */
    int target_number;          /* internal target index(created seq) */
    struct jtag_tap *tap;       /* "-chain-position <tapname|idx>", tap to operate the target */
    int32_t coreid;             /* "-coreid xxx "which device on the TAP? */

    bool defer_examine;
    bool examined;

    /**
     * true if the  target is currently running a downloaded
     * "algorithm" instead of arbitrary user code. OpenOCD code
     * invoking algorithms is trusted to maintain correctness of
     * any cached state (e.g. for flash status), which arbitrary
     * code will have no reason to know about.
     */
    bool running_alg;

    struct target_event_action *event_action;

    int reset_halt;                     /* attempt resetting the CPU into the halted mode? */
    target_addr_t working_area;             /* working area (initialised RAM). Evaluated
                                         * upon first allocation from virtual/physical address. */
    bool working_area_virt_spec;        /* virtual address specified? */
    target_addr_t working_area_virt;    /* specified as "-work-area-virt xxx" */
    bool working_area_phys_spec;        /* physical address specified? */
    target_addr_t working_area_phys;    /* "-work-area-phys xxx" */
    uint32_t working_area_size;         /* "-work-area-size xxx" */
    uint32_t backup_working_area;       /* "-work-area-backup" whether the content of the working area has to be preserved */
    struct working_area *working_areas;/* list of allocated working areas */

    enum target_debug_reason debug_reason;/* reason why the target entered debug state */
    enum target_endianness endianness;  /* "-endian xxx" target endianness */

    /* also see: target_state_name() */
    enum target_state state;            /* the current backend-state (running, halted, ...) */
    struct reg_cache *reg_cache;        /* the first register cache of the target (core regs) */
    struct breakpoint *breakpoints;     /* list of breakpoints */
    struct watchpoint *watchpoints;     /* list of watchpoints */
    struct trace *trace_info;           /* generic trace information */
    struct debug_msg_receiver *dbgmsg;  /* list of debug message receivers */
    uint32_t dbg_msg_enabled;           /* debug message status */
    void *arch_info;                    /* architecture specific information */
    void *private_config;               /* pointer to target specific config data (for jim_configure hook) */
    struct target *next;                /* next target in list */

    bool verbose_halt_msg;              /* display async info in telnet session. Do not display
                                         * lots of halted/resumed info when stepping in debugger. */
    bool halt_issued;                   /* did we transition to halted state? */
    int64_t halt_issued_time;           /* Note time when halt was issued */

                                        /* ARM v7/v8 targets with ADIv5 interface */
    bool dbgbase_set;                   /* By default the debug base is not set */
    uint32_t dbgbase;                   /* Really a Cortex-A specific option, but there is no
                                         * system in place to support target specific options
                                         * currently. */

    bool has_dap;                       /* ADIv5 ? excludes "-chain-position" */
    bool dap_configured;                /* ADIv5 DAP is configured ? */
    bool tap_configured;                /* JTAG tap has been configured through -chain-position? */

    struct rtos *rtos;                  /* Instance of Real Time Operating System support */
    bool rtos_auto_detect;              /* A flag that indicates that the RTOS has been specified as "auto"
                                         * and must be detected when symbols are offered */
    struct backoff_timer backoff;
    int smp;                            /* add some target attributes for smp support */
    struct target_list *head;
    /* the gdb service is there in case of smp, we have only one gdb server
     * for all smp target
     * the target attached to the gdb is changing dynamically by changing
     * gdb_service->target pointer */
    struct gdb_service *gdb_service;

    /* file-I/O information for host to do syscall */
    struct gdb_fileio_info *fileio_info;

    char *gdb_port_override;            /* "-gdb-port xxx" target-specific override for gdb_port */
    int gdb_max_connections;            /* "-gdb-max-connections" */

    struct semihosting *semihosting;
};

/* create by '-event <event> <obj>' option */
struct target_event_action {
    enum target_event event;
    Jim_Interp *interp;
    Jim_Obj *body;
    struct target_event_action *next;
};
```

```c
struct target_type {
    const char *name;

    int (*poll)(struct target *target);
    int (*arch_state)(struct target *target);

    /* target request support */
    int (*target_request_data)(struct target *target, uint32_t size, uint8_t *buffer);

    int (*halt)(struct target *target);
    int (*resume)(struct target *target, int current, target_addr_t address,
            int handle_breakpoints, int debug_execution);
    int (*step)(struct target *target, int current, target_addr_t address,
            int handle_breakpoints);

    int (*assert_reset)(struct target *target);
    int (*deassert_reset)(struct target *target);
    int (*soft_reset_halt)(struct target *target);

    const char *(*get_gdb_arch)(struct target *target);
    int (*get_gdb_reg_list)(struct target *target, struct reg **reg_list[],
            int *reg_list_size, enum target_register_class reg_class);
    int (*get_gdb_reg_list_noread)(struct target *target,
            struct reg **reg_list[], int *reg_list_size,
            enum target_register_class reg_class);

    int (*read_memory)(struct target *target, target_addr_t address,
            uint32_t size, uint32_t count, uint8_t *buffer);
    int (*write_memory)(struct target *target, target_addr_t address,
            uint32_t size, uint32_t count, const uint8_t *buffer);

    int (*read_buffer)(struct target *target, target_addr_t address,
            uint32_t size, uint8_t *buffer);
    int (*write_buffer)(struct target *target, target_addr_t address,
            uint32_t size, const uint8_t *buffer);

    int (*checksum_memory)(struct target *target, target_addr_t address,
            uint32_t count, uint32_t *checksum);
    int (*blank_check_memory)(struct target *target,
            struct target_memory_check_block *blocks, int num_blocks,
            uint8_t erased_value);

    int (*add_breakpoint)(struct target *target, struct breakpoint *breakpoint);
    int (*add_context_breakpoint)(struct target *target, struct breakpoint *breakpoint);
    int (*add_hybrid_breakpoint)(struct target *target, struct breakpoint *breakpoint);
    int (*remove_breakpoint)(struct target *target, struct breakpoint *breakpoint);

    int (*add_watchpoint)(struct target *target, struct watchpoint *watchpoint);
    int (*remove_watchpoint)(struct target *target, struct watchpoint *watchpoint);
    int (*hit_watchpoint)(struct target *target, struct watchpoint **hit_watchpoint);

    /**  use target_run_algorithm().  */
    int (*run_algorithm)(struct target *target, int num_mem_params,
            struct mem_param *mem_params, int num_reg_params,
            struct reg_param *reg_param, target_addr_t entry_point,
            target_addr_t exit_point, int timeout_ms, void *arch_info);
    int (*start_algorithm)(struct target *target, int num_mem_params,
            struct mem_param *mem_params, int num_reg_params,
            struct reg_param *reg_param, target_addr_t entry_point,
            target_addr_t exit_point, void *arch_info);
    int (*wait_algorithm)(struct target *target, int num_mem_params,
            struct mem_param *mem_params, int num_reg_params,
            struct reg_param *reg_param, target_addr_t exit_point,
            int timeout_ms, void *arch_info);

    const struct command_registration *commands;

    int (*target_create)(struct target *target, Jim_Interp *interp);

    int (*target_jim_configure)(struct target *target, Jim_GetOptInfo *goi);

    /* target commands specifically handled by the target */
    /* returns JIM_OK, or JIM_ERR, or JIM_CONTINUE - if option not understood */
    int (*target_jim_commands)(struct target *target, Jim_GetOptInfo *goi);

    /**
     * This method is used to perform target setup that requires
     * JTAG access.
     *
     * This may be called multiple times.  It is called after the
     * scan chain is initially validated, or later after the target
     * is enabled by a JRC.  It may also be called during some
     * parts of the reset sequence.
     *
     * For one-time initialization tasks, use target_was_examined()
     * and target_set_examined().  For example, probe the hardware
     * before setting up chip-specific state, and then set that
     * flag so you don't do that again.
     */
    int (*examine)(struct target *target);

    /* Set up structures for target.
     *
     * It is illegal to talk to the target at this stage as this fn is invoked
     * before the JTAG chain has been examined/verified
     * */
    int (*init_target)(struct command_context *cmd_ctx, struct target *target);

    /**
     * Free all the resources allocated by the target.
     *
     * @param target The target to deinit
     */
    void (*deinit_target)(struct target *target);

    int (*virt2phys)(struct target *target, target_addr_t address, target_addr_t *physical);

    int (*read_phys_memory)(struct target *target, target_addr_t phys_address,
            uint32_t size, uint32_t count, uint8_t *buffer);

    int (*write_phys_memory)(struct target *target, target_addr_t phys_address,
            uint32_t size, uint32_t count, const uint8_t *buffer);

    int (*mmu)(struct target *target, int *enabled);

    /* after reset is complete, the target can check if things are properly set up.
     *
     * This can be used to check if e.g. DCC memory writes have been enabled for
     * arm7/9 targets, which they really should except in the most contrived
     * circumstances.
     */
    int (*check_reset)(struct target *target);

    /* get GDB file-I/O parameters from target
     */
    int (*get_gdb_fileio_info)(struct target *target, struct gdb_fileio_info *fileio_info);

    /* pass GDB file-I/O response to target
     */
    int (*gdb_fileio_end)(struct target *target, int retcode, int fileio_errno, bool ctrl_c);

    /* do target profiling
     */
    int (*profiling)(struct target *target, uint32_t *samples,
            uint32_t max_num_samples, uint32_t *num_samples, uint32_t seconds);

    /* Return the number of address bits this target supports. This will
     * typically be 32 for 32-bit targets, and 64 for 64-bit targets. If not
     * implemented, it's assumed to be 32. */
    unsigned (*address_bits)(struct target *target);
};
```
- `name`
    Identify a type of targets. It's used in "target init" command.

- `poll`
- `arch_state`
    Invoked only from target_arch_state().
    Issue USER() w/architecture specific status.
- `target_request_data`

- `halt`, `resume`, `step`
    target state control

- `assert_reset`, `deassert_reset`, `soft_reset_halt`
    target reset control. assert reset can be invoked when OpenOCD and
    the target is out of sync.
    
    A typical example is that the target was power cycled while OpenOCD
    thought the target was halted or running.
    
    assert_reset() can therefore make no assumptions whatsoever about the
    state of the target
    
    Before assert_reset() for the target is invoked, a TRST/tms and
    chain validation is executed. TRST should not be asserted
    during target assert unless there is no way around it due to
    the way reset's are configured.

    for `deassert_reset`
    The implementation is responsible for polling the
    target such that target->state reflects the
    state correctly.
    
    Otherwise the following would fail, as there will not
    be any "poll" invoked between the "reset run" and
    "halt".
    
    reset run; halt
    
   
- `get_gdb_arch`
    string of target architecture.
- `get_gdb_reg_list`, `get_gdb_reg_list_noread`
    Target register access for GDB.  Do @b not call this function
    directly, use target_get_gdb_reg_list() instead.
    
    Danger! this function will succeed even if the target is running
    and return a register list with dummy values.
    
    The reason is that GDB connection will fail without a valid register
    list, however it is after GDB is connected that monitor commands can
    be run to properly initialize the target
 
    `get_gdb_reg_list_noread` is the Same as get_gdb_reg_list, but doesn't read
    the register values.

- `virt2phys`, `mmu`
	mmu returns if target mmu is enabled(or supported?)
	If no mmu, virt2phys is an identical mapping.
- `read_memory`, `write_memory`, `read_phys_memory`, `write_phys_memory`
    target memory access.
    size: could be 1, 2, 4 bytes.
    count: number of items of <size>

    In phys variants, caches are bypassed and untouched.
    
    If the target does not support disabling caches, leaving them untouched,
    then minimally the actual physical memory location will be read even
    if cache states are unchanged, flushed, etc.
    
    Default implementation is to call read_memory.
    

- `read_buffer`, `write_buffer`

- `checksum_memory`, `blank_check_memory`

- `add_breakpoint`, `add_context_breakpoint`, `add_hybrid_breakpoint`, `remove_breakpoint`
    
    target break-/watchpoint control
    rw: 0 = write, 1 = read, 2 = access
    
    Target must be halted while this is invoked as this
    will actually set up breakpoints on target.
    
    The breakpoint hardware will be set up upon adding the
    first breakpoint.
    
    Upon GDB connection all breakpoints/watchpoints are cleared.
 
    remove breakpoint. hw will only be updated if the target
    is currently halted.
    However, this method can be invoked on unresponsive targets.

- `add_watchpoint`, `remove_watchpoint`, `hit_watchpoint`
    `hit_watchpoint` Find out just hit watchpoint. After the target hits a watchpoint, the
    information could assist gdb to locate where the modified/accessed memory is.

- `run_algorithm`
- `start_algorithm`
- `wait_algorithm`

- `commands`
    
- `target_create`
	Called by "target create" command

- `target_jim_configure`
	Optional. 
    called by "target create" command to parse target specific options.
    returns JIM_CONTINUE - if option not understood in order to fall back to
    common option hanler.

- `target_jim_commands`
- `examine`
- `init_target`
- `deinit_target`
- `check_reset`
- `get_gdb_fileio_info`
- `gdb_fileio_info`
- `gdb_fileio_end`
- `profiling`
- `address_bits`

There is a static array `target_types[]`defining all supported targets.


`target_command_handlers`:

- `targets`
	List all created targets or choose the current target
	(`ctx->current_target` and `ctx->current_target_override`).
- `target`
    - `init`
        invoke proc "init_targets", "init_target_events" and "init_board".
    - `create <name> <type> [<target_options> ...]`
        create a target, and configure it if <target options> provided.
        <name> expands a cmd series like "<name> xxx". So it's saved
		to `target->cmd_name`.
		<type> shall be name of supported target types, like "cortex_m",
		"cortex_a", "aarch64" etc.
    - `current`
		Display current target name
    - `types`
		List supported target types
    - `names`
		List created targets
    - `smp`

- `target create <name> <type> <options>` command handler:
	`target_create` called by `jim_target_create`
	- Parse and check `name` and `type`.
	- Create a target of given type and init default values.
	- Call `target_configure` to handle extra optional cmd options.
		`target->type->target_jim_configure` is called to handled target specific
		options.
	- Call `target->type->target_create`
	- Register commands in `target->type->commands`
	- Register target instance command and `target->type->commands` as subcmds of
		specific target. e.g. If you create a cortex_m target named
		"stm32f1x.cpu", then cortex_m cmds are added and they can also be used as
		subcmds of "stm32f1x.cpu <sub>". Besides there are some common subcmds.

created target commands:
- `<target_name>`
	- `configure`
		further config the target. Use the same option set in "target create"
	- `cget <option>`
		Display value of given option already set to the target
	- `mwd`, `mww`, `mwh`, `mwb`
	- `mdd`, `mdw`, `mdh`, `mdb`
		memory write/display. call `target->type->write_{phys_}memory` and
		`target->type->read_{phys_}memory` respectively
	- `array2mem <var> <width> <addr> <nelem> [phys]`
	- `mem2array <var> <width> <addr> <nelem> [phys]`
		write tcl array of 8/16/32 bit numbers to target memory.
		call `target->type->write_{phys}memory`
	- `eventlist`
	- `curstate`
		Display current target state

	- `arp_examine`
	- `was_examined`
	- `examine_deferred`
	- `arp_halt_gdb`
	- `arp_poll`
	- `arp_reset`
	- `arp_halt`
	- `arp_waitstate`

	- `invoke-event`

	- `target->type->commands` can be used as subcommnads of created target.

- `"target init"`
	- run proc `init_targets`
		Do nothing by default
	- run proc `init_target_events`
	- run proc `init_board`
	- `target_init`
		- call `target_init_one` for each created target
			- `type->init_target`
			- set default type methods
		- target_request_register_commands
		- trace_register_commands
		- register target_exec_command_handlers

`target_exec_command_handlers`
- `fast_load_image`
- `fast_load`
- `profile`
- `virt2phys`
- `reg`
- `poll`
- `wait_halt`
- `halt`
- `resume`
- `reset`
- `soft_reset_halt`
- `step`
- `mdd`, `mdw`, `mdh`, `mdb`, `mwd`, `mww`, `mwh`, `mwb`
- `bp`, `rbp`
- `wp`, `rwp`
- `load_image`, `dump_image`, `verify_image_checksum`, `verify_image`, `test_image`
- `get_reg`, `set_reg`
- `read_memory`, `write_memory`
- `reset_nag`
- `ps`
- `test_mem_access`


##### cortex-a

Following is cortex-a (armv8-a & armv7-a) target type definition.
```c
struct target_type cortexa_target = {
    .name = "cortex_a",

    .poll = cortex_a_poll,
    .arch_state = armv7a_arch_state,

    .halt = cortex_a_halt,
    .resume = cortex_a_resume,
    .step = cortex_a_step,

    .assert_reset = cortex_a_assert_reset,
    .deassert_reset = cortex_a_deassert_reset,

    /* REVISIT allow exporting VFP3 registers ... */
    .get_gdb_arch = arm_get_gdb_arch,
    .get_gdb_reg_list = arm_get_gdb_reg_list,

    .read_memory = cortex_a_read_memory,
    .write_memory = cortex_a_write_memory,

    .read_buffer = cortex_a_read_buffer,
    .write_buffer = cortex_a_write_buffer,

    .checksum_memory = arm_checksum_memory,
    .blank_check_memory = arm_blank_check_memory,

    .run_algorithm = armv4_5_run_algorithm,

    .add_breakpoint = cortex_a_add_breakpoint,
    .add_context_breakpoint = cortex_a_add_context_breakpoint,
    .add_hybrid_breakpoint = cortex_a_add_hybrid_breakpoint,
    .remove_breakpoint = cortex_a_remove_breakpoint,
    .add_watchpoint = NULL,
    .remove_watchpoint = NULL,

    .commands = cortex_a_command_handlers,
    .target_create = cortex_a_target_create,
    .target_jim_configure = adiv5_jim_configure,
    .init_target = cortex_a_init_target,
    .examine = cortex_a_examine,
    .deinit_target = cortex_a_deinit_target,

    .read_phys_memory = cortex_a_read_phys_memory,
    .write_phys_memory = cortex_a_write_phys_memory,
    .mmu = cortex_a_mmu,
    .virt2phys = cortex_a_virt2phys,
};
```

`cortex_a` target accepts extra adiv5 commands to fill following structure.
- `-dap <dapname>`
	The dap through which to access the target
- `-ap-num <num>`
	Access point number
```c
struct adiv5_private_config {
    int ap_num;
    struct adiv5_dap *dap;
};
```

```c
struct cortex_a_common {
    int common_magic;

    /* Context information */
    uint32_t cpudbg_dscr;

    /* Saved cp15 registers */
    uint32_t cp15_control_reg;
    /* latest cp15 register value written and cpsr processor mode */
    uint32_t cp15_control_reg_curr;
    /* auxiliary control reg */
    uint32_t cp15_aux_control_reg;
    /* DACR */
    uint32_t cp15_dacr_reg;
    enum arm_mode curr_mode;

    /* Breakpoint register pairs */
    int brp_num_context;
    int brp_num;
    int brp_num_available;
    struct cortex_a_brp *brp_list;
    int wrp_num;
    int wrp_num_available;
    struct cortex_a_wrp *wrp_list;

    uint32_t cpuid;
    uint32_t didr;

    enum cortex_a_isrmasking_mode isrmasking_mode;
    enum cortex_a_dacrfixup_mode dacrfixup_mode;

    struct armv7a_common armv7a_common;

};
```

```c
```


##### Builtin Commnads

```c
static const struct command_registration command_subcommand_handlers[] = {
    {
        .name = "mode",
        .mode = COMMAND_ANY,
        .jim_handler = jim_command_mode,
        .usage = "[command_name ...]",
        .help = "Returns the command modes allowed by a command: "
            "'any', 'config', or 'exec'. If no command is "
            "specified, returns the current command mode. "
            "Returns 'unknown' if an unknown command is given. "
            "Command can be multiple tokens.",
    },
    COMMAND_REGISTRATION_DONE
};

static const struct command_registration command_builtin_handlers[] = {
    {
        .name = "ocd_find",
        .mode = COMMAND_ANY,
        .jim_handler = jim_find,
        .help = "find full path to file",
        .usage = "file",
    },
    {
        .name = "capture",
        .mode = COMMAND_ANY,
        .jim_handler = jim_capture,
        .help = "Capture progress output and return as tcl return value. If the "
                "progress output was empty, return tcl return value.",
        .usage = "command",
    },
    {
        .name = "echo",
        .handler = jim_echo,
        .mode = COMMAND_ANY,
        .help = "Logs a message at \"user\" priority. "
            "Output message to stdout. "
            "Option \"-n\" suppresses trailing newline",
        .usage = "[-n] string",
    },
    {
        .name = "add_help_text",
        .handler = handle_help_add_command,
        .mode = COMMAND_ANY,
        .help = "Add new command help text; "
            "Command can be multiple tokens.",
        .usage = "command_name helptext_string",
    },
    {
        .name = "add_usage_text",
        .handler = handle_help_add_command,
        .mode = COMMAND_ANY,
        .help = "Add new command usage text; "
            "command can be multiple tokens.",
        .usage = "command_name usage_string",
    },
    {
        .name = "sleep",
        .handler = handle_sleep_command,
        .mode = COMMAND_ANY,
        .help = "Sleep for specified number of milliseconds.  "
            "\"busy\" will busy wait instead (avoid this).",
        .usage = "milliseconds ['busy']",
    },
    {
        .name = "help",
        .handler = handle_help_command,
        .mode = COMMAND_ANY,
        .help = "Show full command help; "
            "command can be multiple tokens.",
        .usage = "[command_name]",
    },
    {
        .name = "usage",
        .handler = handle_help_command,
        .mode = COMMAND_ANY,
        .help = "Show basic command usage; "
            "command can be multiple tokens.",
        .usage = "[command_name]",
    },
    {
        .name = "command",
        .mode = COMMAND_ANY,
        .help = "core command group (introspection)",
        .chain = command_subcommand_handlers,
        .usage = "",
    },
    COMMAND_REGISTRATION_DONE
};
```

##### Server Commnads

- `telnet_port`
    Get/Set telnet port. 4444 by default.

```c
static const struct command_registration telnet_command_handlers[] = {
    {
        .name = "exit",
        .handler = handle_exit_command,
        .mode = COMMAND_EXEC,
        .usage = "",
        .help = "exit telnet session",
    },
    {
        .name = "telnet_port",
        .handler = handle_telnet_port_command,
        .mode = COMMAND_CONFIG,
        .help = "Specify port on which to listen "
            "for incoming telnet connections.  "
            "Read help on 'gdb_port'.",
        .usage = "[port_num]",
    },
    COMMAND_REGISTRATION_DONE
};
```

- `tcl_port`
    Get/Set tcl command port. 7777 by default.

```c
static const struct command_registration tcl_command_handlers[] = {
    {
        .name = "tcl_port",
        .handler = handle_tcl_port_command,
        .mode = COMMAND_CONFIG,
        .help = "Specify port on which to listen "
            "for incoming Tcl syntax.  "
            "Read help on 'gdb_port'.",
        .usage = "[port_num]",
    },
    {
        .name = "tcl_notifications",
        .handler = handle_tcl_notifications_command,
        .mode = COMMAND_EXEC,
        .help = "Target Notification output",
        .usage = "[on|off]",
    },
    {
        .name = "tcl_trace",
        .handler = handle_tcl_trace_command,
        .mode = COMMAND_EXEC,
        .help = "Target trace output",
        .usage = "[on|off]",
    },
    COMMAND_REGISTRATION_DONE
};
```

- `jsp_port`
    Get/Set jsp port. 7777 by default. TODO
```c
static const struct command_registration jsp_command_handlers[] = {
    {
        .name = "jsp_port",
        .handler = handle_jsp_port_command,
        .mode = COMMAND_ANY,
        .help = "Specify port on which to listen "
            "for incoming JSP telnet connections.",
        .usage = "[port_num]",
    },
    COMMAND_REGISTRATION_DONE
};
```

```c
static const struct command_registration server_command_handlers[] = {
    {
        .name = "shutdown",
        .handler = &handle_shutdown_command,
        .mode = COMMAND_ANY,
        .usage = "",
        .help = "shut the server down",
    },
    {
        .name = "poll_period",
        .handler = &handle_poll_period_command,
        .mode = COMMAND_ANY,
        .usage = "",
        .help = "set the servers polling period",
    },
    {
        .name = "bindto",
        .handler = &handle_bindto_command,
        .mode = COMMAND_CONFIG,
        .usage = "[name]",
        .help = "Specify address by name on which to listen for "
            "incoming TCP/IP connections",
    },
    COMMAND_REGISTRATION_DONE
};
```

##### GDB Commands

- `gdb_port`
    Get/Set gdb server port. 3333 by default.

```c
static const struct command_registration gdb_command_handlers[] = {
    {
        .name = "gdb_sync",
        .handler = handle_gdb_sync_command,
        .mode = COMMAND_ANY,
        .help = "next stepi will return immediately allowing "
            "GDB to fetch register state without affecting "
            "target state",
        .usage = ""
    },
    {
        .name = "gdb_port",
        .handler = handle_gdb_port_command,
        .mode = COMMAND_CONFIG,
        .help = "Normally gdb listens to a TCP/IP port. Each subsequent GDB "
            "server listens for the next port number after the "
            "base port number specified. "
            "No arguments reports GDB port. \"pipe\" means listen to stdin "
            "output to stdout, an integer is base port number, \"disabled\" disables "
            "port. Any other string is are interpreted as named pipe to listen to. "
            "Output pipe is the same name as input pipe, but with 'o' appended.",
        .usage = "[port_num]",
    },
    {
        .name = "gdb_memory_map",
        .handler = handle_gdb_memory_map_command,
        .mode = COMMAND_CONFIG,
        .help = "enable or disable memory map",
        .usage = "('enable'|'disable')"
    },
    {
        .name = "gdb_flash_program",
        .handler = handle_gdb_flash_program_command,
        .mode = COMMAND_CONFIG,
        .help = "enable or disable flash program",
        .usage = "('enable'|'disable')"
    },
    {
        .name = "gdb_report_data_abort",
        .handler = handle_gdb_report_data_abort_command,
        .mode = COMMAND_CONFIG,
        .help = "enable or disable reporting data aborts",
        .usage = "('enable'|'disable')"
    },
    {
        .name = "gdb_report_register_access_error",
        .handler = handle_gdb_report_register_access_error,
        .mode = COMMAND_CONFIG,
        .help = "enable or disable reporting register access errors",
        .usage = "('enable'|'disable')"
    },
    {
        .name = "gdb_breakpoint_override",
        .handler = handle_gdb_breakpoint_override_command,
        .mode = COMMAND_ANY,
        .help = "Display or specify type of breakpoint "
            "to be used by gdb 'break' commands.",
        .usage = "('hard'|'soft'|'disable')"
    },
    {
        .name = "gdb_target_description",
        .handler = handle_gdb_target_description_command,
        .mode = COMMAND_CONFIG,
        .help = "enable or disable target description",
        .usage = "('enable'|'disable')"
    },
    {
        .name = "gdb_save_tdesc",
        .handler = handle_gdb_save_tdesc_command,
        .mode = COMMAND_EXEC,
        .help = "Save the target description file",
        .usage = "",
    },
    COMMAND_REGISTRATION_DONE
};
```

##### Log Commands

```c
static const struct command_registration log_command_handlers[] = {
    {
        .name = "log_output",
        .handler = handle_log_output_command,
        .mode = COMMAND_ANY,
        .help = "redirect logging to a file (default: stderr)",
        .usage = "[file_name | \"default\"]",
    },
    {
        .name = "debug_level",
        .handler = handle_debug_level_command,
        .mode = COMMAND_ANY,
        .help = "Sets the verbosity level of debugging output. "
            "0 shows errors only; 1 adds warnings; "
            "2 (default) adds other info; 3 adds debugging; "
            "4 adds extra verbose debugging.",
        .usage = "number",
    },
    COMMAND_REGISTRATION_DONE
};
```

##### RTT Commands

```c
static const struct command_registration rtt_server_subcommand_handlers[] = {
    {
        .name = "start",
        .handler = handle_rtt_start_command,
        .mode = COMMAND_ANY,
        .help = "Start a RTT server",
        .usage = "<port> <channel>"
    },
    {
        .name = "stop",
        .handler = handle_rtt_stop_command,
        .mode = COMMAND_ANY,
        .help = "Stop a RTT server",
        .usage = "<port>"
    },
    COMMAND_REGISTRATION_DONE
};

static const struct command_registration rtt_server_command_handlers[] = {
    {
        .name = "server",
        .mode = COMMAND_ANY,
        .help = "RTT server",
        .usage = "",
        .chain = rtt_server_subcommand_handlers
    },
    COMMAND_REGISTRATION_DONE
};

static const struct command_registration rtt_command_handlers[] = {
    {
        .name = "rtt",
        .mode = COMMAND_ANY,
        .help = "RTT",
        .usage = "",
        .chain = rtt_server_command_handlers
    },
    COMMAND_REGISTRATION_DONE
};
```

##### Transport Commands

```c
static const struct command_registration transport_commands[] = {
    {
        .name = "init",
        .handler = handle_transport_init,
        /* this would be COMMAND_CONFIG ... except that
         * it needs to trigger event handlers that may
         * require COMMAND_EXEC ...
         */
        .mode = COMMAND_ANY,
        .help = "Initialize this session's transport",
        .usage = ""
    },
    {
        .name = "list",
        .handler = handle_transport_list,
        .mode = COMMAND_ANY,
        .help = "list all built-in transports",
        .usage = ""
    },
    {
        .name = "select",
        .jim_handler = jim_transport_select,
        .mode = COMMAND_ANY,
        .help = "Select this session's transport",
        .usage = "[transport_name]",
    },
    COMMAND_REGISTRATION_DONE
};

static const struct command_registration transport_group[] = {
    {
        .name = "transport",
        .mode = COMMAND_ANY,
        .help = "Transport command group",
        .chain = transport_commands,
        .usage = ""
    },
    COMMAND_REGISTRATION_DONE
};
```

##### Server Services
Usually openocd creates serveral listening ports to provide console to control
the adapter or debugger.

```c
struct service {
    char *name;

    /* port set string, and number. type is CONNECTION_TCP for numbered port
     * or CONNECTION_PIPE for "pipe" port */
    enum connection_type type;
    char *port;
    unsigned short portnumber;

    int fd;     /* usually the socket fd */
    struct sockaddr_in sin;
    int max_connections;
    struct connection *connections;     /* connections of the service */
    new_connection_handler_t new_connection;    /* called when creating new connection */
    input_handler_t input;
    connection_closed_handler_t connection_closed;
    void *priv;
    struct service *next;   /* linked by "services" */
};
```

Multiple connections may be established on different FDs for a service.
```c
struct connection {
    int fd;
    int fd_out; /* When using pipes we're writing to a different fd */
    struct sockaddr_in sin;
    struct command_context *cmd_ctx;
    struct service *service;
    bool input_pending;
    void *priv;
    struct connection *next;    /* link other connections for this service */
};
```

`add_service` creates a service and listen on it. No connections are established
yet. After entering `server_loop`, new connection is added if someone is
connecting to the listen port.
```c
int add_service(char *name,
    const char *port,
    int max_connections,
    new_connection_handler_t new_connection_handler,
    input_handler_t input_handler,
    connection_closed_handler_t connection_closed_handler,
    void *priv)
{
    struct service *c, **p;
    struct hostent *hp;
    int so_reuseaddr_option = 1;

    c = malloc(sizeof(struct service));

    c->name = strdup(name);
    c->port = strdup(port);
    c->max_connections = 1; /* Only TCP/IP ports can support more than one connection */
    c->fd = -1;
    c->connections = NULL;
    c->new_connection = new_connection_handler;
    c->input = input_handler;
    c->connection_closed = connection_closed_handler;
    c->priv = priv;
    c->next = NULL;
    long portnumber;
    if (strcmp(c->port, "pipe") == 0)
        c->type = CONNECTION_STDINOUT;
    else {
        char *end;
        portnumber = strtol(c->port, &end, 0);
        if (!*end && (parse_long(c->port, &portnumber) == ERROR_OK)) {
            c->portnumber = portnumber;
            c->type = CONNECTION_TCP;
        } else
            c->type = CONNECTION_PIPE;
    }

    if (c->type == CONNECTION_TCP) {
        c->max_connections = max_connections;

        c->fd = socket(AF_INET, SOCK_STREAM, 0);
        if (c->fd == -1) {
            LOG_ERROR("error creating socket: %s", strerror(errno));
            free_service(c);
            return ERROR_FAIL;
        }

        setsockopt(c->fd,
            SOL_SOCKET,
            SO_REUSEADDR,
            (void *)&so_reuseaddr_option,
            sizeof(int));

        socket_nonblock(c->fd);

        memset(&c->sin, 0, sizeof(c->sin));
        c->sin.sin_family = AF_INET;

        if (bindto_name == NULL)
            c->sin.sin_addr.s_addr = htonl(INADDR_LOOPBACK);
        else {
            hp = gethostbyname(bindto_name);
            if (hp == NULL) {
                LOG_ERROR("couldn't resolve bindto address: %s", bindto_name);
                close_socket(c->fd);
                free_service(c);
                return ERROR_FAIL;
            }
            memcpy(&c->sin.sin_addr, hp->h_addr_list[0], hp->h_length);
        }
        c->sin.sin_port = htons(c->portnumber);

        if (bind(c->fd, (struct sockaddr *)&c->sin, sizeof(c->sin)) == -1) {
            LOG_ERROR("couldn't bind %s to socket on port %d: %s", name, c->portnumber, strerror(errno));
            close_socket(c->fd);
            free_service(c);
            return ERROR_FAIL;
        }

#ifndef _WIN32
        int segsize = 65536;
        setsockopt(c->fd, IPPROTO_TCP, TCP_MAXSEG,  &segsize, sizeof(int));
#endif
        int window_size = 128 * 1024;

        /* These setsockopt()s must happen before the listen() */

        setsockopt(c->fd, SOL_SOCKET, SO_SNDBUF,
            (char *)&window_size, sizeof(window_size));
        setsockopt(c->fd, SOL_SOCKET, SO_RCVBUF,
            (char *)&window_size, sizeof(window_size));

        if (listen(c->fd, 1) == -1) {
            LOG_ERROR("couldn't listen on socket: %s", strerror(errno));
            close_socket(c->fd);
            free_service(c);
            return ERROR_FAIL;
        }

        struct sockaddr_in addr_in;
        addr_in.sin_port = 0;
        socklen_t addr_in_size = sizeof(addr_in);
        if (getsockname(c->fd, (struct sockaddr *)&addr_in, &addr_in_size) == 0)
            LOG_INFO("Listening on port %hu for %s connections",
                 ntohs(addr_in.sin_port), name);
    } else if (c->type == CONNECTION_STDINOUT) {
        c->fd = fileno(stdin);

#ifdef _WIN32
        /* for win32 set stdin/stdout to binary mode */
        if (_setmode(_fileno(stdout), _O_BINARY) < 0)
            LOG_WARNING("cannot change stdout mode to binary");
        if (_setmode(_fileno(stdin), _O_BINARY) < 0)
            LOG_WARNING("cannot change stdin mode to binary");
        if (_setmode(_fileno(stderr), _O_BINARY) < 0)
            LOG_WARNING("cannot change stderr mode to binary");
#else
        socket_nonblock(c->fd);
#endif
    } else if (c->type == CONNECTION_PIPE) {
#ifdef _WIN32
        /* we currently do not support named pipes under win32
         * so exit openocd for now */
        LOG_ERROR("Named pipes currently not supported under this os");
        free_service(c);
        return ERROR_FAIL;
#else
        /* Pipe we're reading from */
        c->fd = open(c->port, O_RDONLY | O_NONBLOCK);
        if (c->fd == -1) {
            LOG_ERROR("could not open %s", c->port);
            free_service(c);
            return ERROR_FAIL;
        }
#endif
    }

    /* add to the end of linked list */
    for (p = &services; *p; p = &(*p)->next)
        ;
    *p = c;

    return ERROR_OK;
}
```
`add_connection` adds a connection for a service.
```c
static int add_connection(struct service *service, struct command_context *cmd_ctx)
{
    socklen_t address_size;
    struct connection *c, **p;
    int retval;
    int flag = 1;

    c = malloc(sizeof(struct connection));
    c->fd = -1;
    c->fd_out = -1;
    memset(&c->sin, 0, sizeof(c->sin));
    c->cmd_ctx = copy_command_context(cmd_ctx);
    c->service = service;
    c->input_pending = false;
    c->priv = NULL;
    c->next = NULL;

    if (service->type == CONNECTION_TCP) {
        address_size = sizeof(c->sin);

        c->fd = accept(service->fd, (struct sockaddr *)&service->sin, &address_size);
        c->fd_out = c->fd;

        /* This increases performance dramatically for e.g. GDB load which
         * does not have a sliding window protocol.
         *
         * Ignore errors from this fn as it probably just means less performance
         */
        setsockopt(c->fd,   /* socket affected */
            IPPROTO_TCP,            /* set option at TCP level */
            TCP_NODELAY,            /* name of option */
            (char *)&flag,          /* the cast is historical cruft */
            sizeof(int));           /* length of option value */

        LOG_INFO("accepting '%s' connection on tcp/%s", service->name, service->port);
        retval = service->new_connection(c);
        if (retval != ERROR_OK) {
            close_socket(c->fd);
            LOG_ERROR("attempted '%s' connection rejected", service->name);
            command_done(c->cmd_ctx);
            free(c);
            return retval;
        }
    } else if (service->type == CONNECTION_STDINOUT) {
        c->fd = service->fd;
        c->fd_out = fileno(stdout);

#ifdef _WIN32
        /* we are using stdin/out so ignore ctrl-c under windoze */
        SetConsoleCtrlHandler(NULL, TRUE);
#endif

        /* do not check for new connections again on stdin */
        service->fd = -1;

        LOG_INFO("accepting '%s' connection from pipe", service->name);
        retval = service->new_connection(c);
        if (retval != ERROR_OK) {
            LOG_ERROR("attempted '%s' connection rejected", service->name);
            command_done(c->cmd_ctx);
            free(c);
            return retval;
        }
    } else if (service->type == CONNECTION_PIPE) {
        c->fd = service->fd;
        /* do not check for new connections again on stdin */
        service->fd = -1;

        char *out_file = alloc_printf("%so", service->port);
        c->fd_out = open(out_file, O_WRONLY);
        free(out_file);
        if (c->fd_out == -1) {
            LOG_ERROR("could not open %s", service->port);
            command_done(c->cmd_ctx);
            free(c);
            return ERROR_FAIL;
        }

        LOG_INFO("accepting '%s' connection from pipe %s", service->name, service->port);
        retval = service->new_connection(c);
        if (retval != ERROR_OK) {
            LOG_ERROR("attempted '%s' connection rejected", service->name);
            command_done(c->cmd_ctx);
            free(c);
            return retval;
        }
    }

    /* add to the end of linked list */
    for (p = &service->connections; *p; p = &(*p)->next)
        ;
    *p = c;

    if (service->max_connections != CONNECTION_LIMIT_UNLIMITED)
        service->max_connections--;

    return ERROR_OK;
}
```

```c
```

## Flash

Openocd first writes a small code snippet called flash writer to target ram
and invoke it to write transmitted data to nor or nand flash through chip
specific flash controllers or spi controllers.

### Nor Flash

```c
struct flash_driver {
    const char *name;
    const char *usage;

    const struct command_registration *commands;	/* driver specific */

    __FLASH_BANK_COMMAND((*flash_bank_command));

    int (*erase)(struct flash_bank *bank, unsigned int first,
        unsigned int last);

    int (*protect)(struct flash_bank *bank, int set, unsigned int first,
        unsigned int last);

    int (*write)(struct flash_bank *bank,
            const uint8_t *buffer, uint32_t offset, uint32_t count);

     int (*read)(struct flash_bank *bank,
            uint8_t *buffer, uint32_t offset, uint32_t count);

    int (*verify)(struct flash_bank *bank,
            const uint8_t *buffer, uint32_t offset, uint32_t count);

    int (*probe)(struct flash_bank *bank);

    int (*erase_check)(struct flash_bank *bank);

    int (*protect_check)(struct flash_bank *bank);

    int (*info)(struct flash_bank *bank, struct command_invocation *cmd);

    int (*auto_probe)(struct flash_bank *bank);

    void (*free_driver_priv)(struct flash_bank *bank);
};
```
- `erase`
    Bank/sector erase routine (target-specific).  When
    called, the flash driver should erase the specified sectors
    using whatever means are at its disposal.
    
    @param bank The bank of flash to be erased.
    @param first The number of the first sector to erase, typically 0.
    @param last The number of the last sector to erase, typically N-1.
    @returns ERROR_OK if successful; otherwise, an error code.
    
- `protect`
    When called, the driver should enable/disable protection
    for MINIMUM the range covered by first..last sectors
    inclusive. Some chips have alignment requirements will
    cause the actual range to be protected / unprotected to
    be larger than the first..last range.
    
    @param bank The bank to protect or unprotect.
    @param set If non-zero, enable protection; if 0, disable it.
    @param first The first sector to (un)protect, typically 0.
    @param last The last sector to (un)project, typically N-1.
    @returns ERROR_OK if successful; otherwise, an error code.

- `write`
    Program data into the flash.  Note CPU address will be
    "bank->base + offset", while the physical address is
    dependent upon current target MMU mappings.
    
    @param bank The bank to program
    @param buffer The data bytes to write.
    @param offset The offset into the chip to program.
    @param count The number of bytes to write.
    @returns ERROR_OK if successful; otherwise, an error code.

- `read`
    Read data from the flash. Note CPU address will be
    "bank->base + offset", while the physical address is
    dependent upon current target MMU mappings.
    
    @param bank The bank to read.
    @param buffer The data bytes read.
    @param offset The offset into the chip to read.
    @param count The number of bytes to read.
    @returns ERROR_OK if successful; otherwise, an error code.
    
- `verify`
    Verify data in flash.  Note CPU address will be
    "bank->base + offset", while the physical address is
    dependent upon current target MMU mappings.
    
    @param bank The bank to verify
    @param buffer The data bytes to verify against.
    @param offset The offset into the chip to verify.
    @param count The number of bytes to verify.
    @returns ERROR_OK if successful; otherwise, an error code.

- `probe`
    Probe to determine what kind of flash is present.
    This is invoked by the "probe" script command.
    
    @param bank The bank to probe
    @returns ERROR_OK if successful; otherwise, an error code.
    
- `erase_check`
    Check the erasure status of a flash bank.
    When called, the driver routine must perform the required
    checks and then set the @c flash_sector_s::is_erased field
    for each of the flash banks's sectors.
    
    @param bank The bank to check
    @returns ERROR_OK if successful; otherwise, an error code.
    
- `protect_check`
    Determine if the specific bank is "protected" or not.
    When called, the driver routine must must perform the
    required protection check(s) and then set the @c
    flash_sector_s::is_protected field for each of the flash
    bank's sectors.
    
    If protection is not implemented, set method to NULL
    
    @param bank - the bank to check
    @returns ERROR_OK if successful; otherwise, an error code.
    
- `info`
    Display human-readable information about the flash
    bank.
    
    @param bank - the bank to get info about
    @param cmd - command invocation instance for which to generate
                 the textual output
    @returns ERROR_OK if successful; otherwise, an error code.
    
- `auto_probe`
    A more gentle flavor of flash_driver_s::probe, performing
    setup with less noise.  Generally, driver routines should test
    to see if the bank has already been probed; if it has, the
    driver probably should not perform its probe a second time.
    
    This callback is often called from the inside of other
    routines (e.g. GDB flash downloads) to autoprobe the flash as
    it is programming the flash.
    
    @param bank - the bank to probe
    @returns ERROR_OK if successful; otherwise, an error code.
    
- `free_driver_priv`
    Deallocates private driver structures.
    Use default_flash_free_driver_priv() to simply free(bank->driver_priv)
    
    @param bank - the bank being destroyed
    


```c
struct flash_sector {
    uint32_t offset; /** Bus offset from start of the flash chip (in bytes). */
    uint32_t size; /** Number of bytes in this flash sector. */

    /**
     * Indication of erasure status: 0 = not erased, 1 = erased,
     * other = unknown.  Set by @c flash_driver_s::erase_check only.
     *
     * This information must be considered stale immediately.
     * Don't set it in flash_driver_s::erase or a device mass_erase
     * Don't clear it in flash_driver_s::write
     * The flag is not used in a protection block
     */
    int is_erased;

    /**
     * Indication of protection status: 0 = unprotected/unlocked,
     * 1 = protected/locked, other = unknown.  Set by
     * @c flash_driver_s::protect_check.
     *
     * This information must be considered stale immediately.
     * A million things could make it stale: power cycle,
     * reset of target, code running on target, etc.
     *
     * If a flash_bank uses an extra array of protection blocks,
     * protection flag is not valid in sector array
     */
    int is_protected;
};
```
   
   

```c
struct flash_bank {
    char *name;

    struct target *target; /**< Target to which this bank belongs. */

    const struct flash_driver *driver; /**< Driver for this bank. */
    void *driver_priv; /**< Private driver storage pointer */

    unsigned int bank_number; /**< The 'bank' (or chip number) of this instance. */
    target_addr_t base; /**< The base address of this bank */
    uint32_t size; /**< The size of this chip bank, in bytes */

    unsigned int chip_width; /**< Width of the chip in bytes (1,2,4 bytes) */
    unsigned int bus_width; /**< Maximum bus width, in bytes (1,2,4 bytes) */

    uint8_t erased_value;	/** Erased value. Defaults to 0xFF. */
    uint8_t default_padded_value;

    /** Required alignment of flash write start address.
     * Default 0, no alignment. Can be any power of two or FLASH_WRITE_ALIGN_SECTOR */
    uint32_t write_start_alignment;
    /** Required alignment of flash write end address.
     * Default 0, no alignment. Can be any power of two or FLASH_WRITE_ALIGN_SECTOR */
    uint32_t write_end_alignment;
    /** Minimal gap between sections to discontinue flash write
     * Default FLASH_WRITE_GAP_SECTOR splits the write if one or more untouched
     * sectors in between.
     * Can be size in bytes or FLASH_WRITE_CONTINUOUS */
    uint32_t minimal_write_gap;

    /**
     * The number of sectors on this chip.  This value will
     * be set initially to 0, and the flash driver must set this to
     * some non-zero value during "probe()" or "auto_probe()".
     */
    unsigned int num_sectors;
    /** Array of sectors, allocated and initialized by the flash driver */
    struct flash_sector *sectors;

    /**
     * The number of protection blocks in this bank. This value
     * is set initially to 0 and sectors are used as protection blocks.
     * Driver probe can set protection blocks array to work with
     * protection granularity different than sector size.
     */
    unsigned int num_prot_blocks;
    /** Array of protection blocks, allocated and initialized by the flash driver */
    struct flash_sector *prot_blocks;

    struct flash_bank *next; /**< The next flash bank on this chip. linked by flash_banks*/
};
```

##### Commands


- `flash`
	Following 4 subcommands are used to config flash.
	- `bank <name> <driver> <base> <size> <chip_width> <bus_width> <target> [options]`
		Add a flash bank to specified target.
	- `init`
		If flash bank exists, register `flash_exec_command_handlers` to
		support further operations.
	- `banks`
		Display all flash banks' characters.
	- `list`
		Returns a list of details about the flash banks
	
	- `probe <bank_id>`
		Call bank->driver->probe to init base, size, sectors, priv data etc.
	- `info <bank_id> [sectors]`
		Show location and protection state of each section.
		bank->driver->info may prints more.
	- `erase_check <bank_id>`
		Call bank->driver->erase_check to check erase state. List list states
		of all sectors.
	- `erase_sector <bank_id> <first_sector> (<last_sector_num>|last)`
		Erase a range of sectors in a flash bank.
		Invoke `bank->driver->erase` to do it.
	- `erase_address`
	- `filld`, `fillw`, `fillh`, `fillb`
	- `mdb`, `mdh`, `mdw`
	- `write_bank`
	- `write_image [erase] [unlock] <filename> [offset [file_type]]`
		Write an image to flash, optionally first unportect and/or erase the
		region to be used. Allow optional offset from beginning of bank.
	- `verify_image <filename> [offset [file_type]]`
		Just verify written image against given file.
	- `read_bank`
	- `verify_bank`
	- `protect`
	- `padded_value`

- `flash_bank_command`
    Finish the "flash bank" command for @a bank.  The
    @a bank parameter will have been filled in by the core flash
    layer when this routine is called, and the driver can store
    additional information in its struct flash_bank::driver_priv field.
    
    The CMD_ARGV are: @par
    @code
    CMD_ARGV[0] = bank
    CMD_ARGV[1] = drivername {name above}
    CMD_ARGV[2] = baseaddress
    CMD_ARGV[3] = lengthbytes
    CMD_ARGV[4] = chip_width_in bytes
    CMD_ARGV[5] = bus_width_in_bytes
    CMD_ARGV[6] = driver-specific parameters
    @endcode
    
    For example, CMD_ARGV[4] = 2 (for 16 bit flash),
     CMD_ARGV[5] = 4 (for 32 bit bus).
    
    If extra arguments are provided (@a CMD_ARGC > 6), they will
    start in @a CMD_ARGV[6].  These can be used to implement
    driver-specific extensions.
    
    @returns ERROR_OK if successful; otherwise, an error code.
    

```c
```

## RTT

```c
```

## Init Flow

- constructors
Following transports are registered in constructors.
- `jtag_transport`
- `swim_transport`
- `swd_transport`
- `dapdirect_jtag_transport`
- `dapdirect_swd_transport`
- `hl_swd_transport`
- `hl_jtag_transport`

- `openocd_main`:
	- `setup_command_handler` establishes a command execution environment.
		- `command_init`
			Create interpreter and command context(the `global_cmd_ctx`).
			Register builtin commands.

			Evaluate startup.tcl. startup.tcl is a collection of startup.tcl files
			in many module directories. It defines useful procs invoking submodule commands.

		- Call command registrants of each module.

	- `util_init`
		register util commands. Only a "ms" returning current time in ms.
	- `rtt_init`
		TODO

	- Enter `COMMAND_CONFIG` mode

	- `openocd_thread`
		- `parse_cmdline_args`
			Use one or more `-f <script>` to add commands to source user cfgs
			Use `-s <dir>` to add cfg script searching path
			Use `-d <num>` to set log debug level
			Use `-l <file/stderr>` to set log file
			Use one or more `-c <cmd>` to add more commands.
		- `server_preinit`
			Set sighandlers.
		- `parse_config_file`
			Execute commands from `-f/-c` options. (User cfg are evaluated)
			User cfg should include following commands:
			- `adapter driver <xxx>`
				Select adapter driver and register adapter cmds
			- optional reset, speed, serial cfgs
			- `transport select <xxx>`
			- `jtag newtap <xxx>` or `swd newdap <name>`
			- `target create <name> <type> <options>`
				
		- `server_init`
			- `tcl_init`
				Add tcl service
			- `telnet_init`
		- run "init" command if "noinit" has not executed in user cfg or
			commands. Duplicated invocation to "init" are ignored.
			- eval "target init"
			- `adapter_init` in turn calls `adapter->init` and set speed.
			- Change command context from "config" to "exec" temperarily for
			   following "transport init"
			- "transport init"
			- "dap init"
			- `target_examine`
			- Change command context back to "config"
			- "flash init"
			- "nand init"
			- "pld init"
			- Finally change command context to "exec"
			- "tpiu init"
			- `gdb_target_add_all`
			-  eval "_run_post_init_commands"

		- `server_loop`
			loops until `shutdown_openocd`  is not equal to `CONTINUE_MAIN_LOOP`.
			(i.e. on receiving term signals)
	
	- clean up resources

What should user script configure?
1. Select physical adapter and transport by "adapter driver <xxx>" and
    "transport select <jtag|swd>"
2. Create dap by "jtag newtap <chip> <tag> <options>" or "swd newdap <chip> <tag>"

In some use cases, we just want to execute several commands, e.g. to flash an
image. We could invoke openocd with `-f cfg` and several `-c cmd`. The last
command should be "exit" if we needn't enter into openocd server loop.

```c
```
