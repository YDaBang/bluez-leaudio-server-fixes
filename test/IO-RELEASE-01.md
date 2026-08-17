# BAP/USR/SCC/IO-RELEASE-01

Reproduction case added to `unit/test-bap.c`.  Without patch 223 this
hangs: the Codec Configured notification never arrives.

Register it with `test_server_state`, not `test_server`.

```c
nsition immediately. With an IO attached the server can only leave
 * Releasing through the IO disconnect handler, so this exercises the branch the
 * rest of the suite never reaches.
 */
static void state_io_release(struct bt_bap_stream *stream,
					uint8_t old_state, uint8_t new_state,
					void *user_data)
{
	static int io_fds[2] = { -1, -1 };

	switch (new_state) {
	case BT_BAP_STREAM_STATE_STREAMING:
		if (io_fds[0] < 0)
			g_assert(!socketpair(AF_UNIX,
					SOCK_SEQPACKET | SOCK_CLOEXEC, 0,
					io_fds));

		g_assert(bt_bap_stream_set_io(stream, io_fds[0]));
		break;
	}
}

static struct test_config cfg_src_io_release = {
	.cc = LC3_CONFIG_16_2,
	.qos = LC3_QOS_16_2_1,
	.src = true,
	.state = BT_BAP_STREAM_STATE_STREAMING,
	.state_func = state_io_release,
};

/* The ASE has to notify Releasing (0x06) and then a settled state. Codec
 * Configured (0x01) is what stream_io_disconnected() selects.
 */
#define ASE_SRC_RELEASE_SETTLES \
	IOV_DATA(0x52, 0x22, 0x00, 0x08, 0x01, 0x03), \
	IOV_DATA(0x1b, 0x22, 0x00, 0x08, 0x01, 0x03, 0x00, 0x00), \
	IOV_NULL, \
	IOV_DATA(0x1b, 0x1c, 0x00, 0x03, 0x06), \
	IOV_NULL, \
	IOV_DATA(0x1b, 0x1c, 0x00, 0x03, 0x01, 0x00, 0x02, 0x01, \
			0x0a, 0x00, 0x20, 0x4e, 0x00, 0x40, 0x9c, 0x00, \
			0x20, 0x4e, 0x00, 0x40, 0x9c, 0x00, 0x06, 0x00, \
			0x00, 0x00, 0x00, 0x0a, 0x02, 0x01, 0x03, 0x02, \
			0x02, 0x01, 0x03, 0x04, 0x28, 0x00)

#define SCC_SRC_IO_RELEASE \
	SCC_SRC_ENABLE, \
	SRC_START, \
	ASE_SRC_RELEASE_SETTLES

static void state_disable_release(struct bt_bap_stream *stream,
					uint8_t old_state, uint8_t new_state,
					void *user_data)
{
	struct test_data *data = user_data;
	uint8_t id;

	switch (new_state) {
	case BT_BAP_STREAM_STATE_DISABLING:
		id = bt_bap_stream_release(stream, bap_release, data);
		g_assert(id);
		break;
	}
}

static struct test_config cfg_src_disable_release = {
	.cc = LC3_CONFIG_16_2,
	.qos = LC3_QOS_16_2_1,
	.src = true,
	.state = BT_BAP_STREAM_STATE_DISABLING,
	.state_func = state_disable_release,
};

#define SCC_SRC_DISABLE_RELEASE \
	SCC_SRC_DISABLE, \
	ASE_SRC_RELEASE

/* Test Purpose:
 * Verify that a Unicast Client IUT can release an ASE by initiating a Release
 * operation.
 *
 * Pass verdict:
 * The IUT successfully writes to the ASE Control Point characteristic with the
 * opcode set to 0x08 (Release) and the specified parameters.
 */
static void test_ucl_scc_release(void)
{
	define_test("BAP/UCL/SCC/BV-106-C [UCL SNK Release in Codec Configured"
			" state]",
			test_setup, test_client, &cfg_src_cc_release,
			SCC_SRC_CC_RELEASE);
	define_test("BAP/UCL/SCC/BV-107-C [UCL SRC Release in Codec Configured"
			" state]",
			test_setup, test_client, &cfg_snk_cc_release,
			SCC_SNK_CC_RELEASE);
	define_test("BAP/UCL/SCC/BV-108-C [UCL SNK Release in QoS Configured"
			" state]",
			test_setup, test_client, &cfg_src_qos_release,
			SCC_SRC_QOS_RELEASE);
	define_test("BAP/UCL/SCC/BV-109-C [UCL SRC Release in QoS Configured"
			" state]",
			test_setup, test_client, &cfg_snk_qos_release,
			SCC_SNK_QOS_RELEASE);
	define_test("BAP/UCL/SCC/BV-110-C [UCL SNK Release in Enabling state]",
			test_setup, test_client, &cfg_src_enable_release,
			SCC_SRC_ENABLE_RELEASE);
	define_test("BAP/UCL/SCC/BV-111-C [UCL SRC Release in Enabling or"
			" Streaming state]",
			test_setup, test_client, &cfg_snk_enable_release,
			SCC_SNK_ENABLE_RELEASE);
	define_test("BAP/UCL/SCC/BV-112-C [UCL SNK Release in Streaming state]",
			test_setup, test_client, &cfg_src_start_release,
```
