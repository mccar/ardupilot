ROOT = $(dir $(lastword $(MAKEFILE_LIST)))/../

WAF_BINARY = $(realpath $(ROOT)/modules/waf/waf-light)
WAF = python $(WAF_BINARY) $(WAF_FLAGS)

EXPLICIT_COMMANDS = check check-all clean list_boards

VEHICLES = copter plane rover sub heli

BOARD_LIST := $(shell $(WAF) list_boards | head -1)

SITL_ROOT ?= $(ROOT)custom/sitl
NAME ?= plane
TYPE ?= plane
COPTER_TYPES = quad hexa octa octa-quad tri y6 heli heli-dual
ROVER_TYPES = rover balancebot sailboat
SUB_TYPES = vectored vectored_6dof
VEHICLE ?= $(if $(filter $(COPTER_TYPES),$(TYPE)),ArduCopter,$(if $(filter $(ROVER_TYPES),$(TYPE)),Rover,$(if $(filter $(SUB_TYPES),$(TYPE)),ArduSub,ArduPlane)))
INSTANCE ?= 0
BIND_ADDR ?= 0.0.0.0
TCP_PORT ?= 5765
QGC_HOST ?= 127.0.0.1
UDP_PORT ?= 14550
SITL_EXTRA_ARGS ?=
MAVPROXY_ARGS ?= --out=tcpin:$(BIND_ADDR):$(TCP_PORT) --out=udp:$(QGC_HOST):$(UDP_PORT)
SITL_DIR = $(abspath $(SITL_ROOT)/$(NAME))
SITL_PARAM_FILE ?= $(SITL_DIR)/$(NAME).parm
SITL_PID_FILE = $(SITL_DIR)/sim_vehicle.pid
SITL_LOG_FILE = $(SITL_DIR)/sim_vehicle.log
SITL_PARAM_ARG = $(if $(SITL_PARAM_FILE),--add-param-file $(SITL_PARAM_FILE),)

all: help

$(WAF_BINARY):
	@git submodule init && git submodule update

waf-%: $(WAF_BINARY)
	@$(WAF) $*

%-configure: $(WAF_BINARY)
	@$(WAF) configure --board $*

$(EXPLICIT_COMMANDS): $(WAF_BINARY)
	@$(WAF) $@

$(VEHICLES): $(WAF_BINARY)
	@echo Build for vehicle $@
	@$(WAF) $@

.DEFAULT: %-configure
	@$(WAF) configure --board $@ build

define target_template
$(1)-$(2) : $(1)-configure $(2)
endef

$(foreach board,$(BOARD_LIST),$(foreach vehicle,$(VEHICLES),$(eval $(call target_template,$(board),$(vehicle)))))

sitl-check-name:
	@case "$(NAME)" in ""|.*|*..*|*[!A-Za-z0-9_.-]*) \
		echo "Invalid NAME='$(NAME)'. Use letters, numbers, dot, underscore, or dash."; \
		exit 2;; \
	esac

sitl-create: sitl-check-name
	@mkdir -p "$(SITL_DIR)"
	@if [ -n "$(SITL_PARAM_FILE)" ] && [ ! -f "$(SITL_PARAM_FILE)" ]; then \
		: > "$(SITL_PARAM_FILE)"; \
	fi
	@echo "SITL profile: $(SITL_DIR)"
	@echo "Vehicle: $(VEHICLE), type/frame: $(TYPE), instance: $(INSTANCE)"
	@ip=$$(hostname -I 2>/dev/null | awk '{print $$1}'); \
	if [ -z "$$ip" ]; then ip=127.0.0.1; fi; \
	echo "MAVSDK TCP: tcp://$$ip:$(TCP_PORT)"
	@echo "QGroundControl UDP: $(QGC_HOST):$(UDP_PORT)"

sitl-run: sitl-create
	@if [ -f "$(SITL_PID_FILE)" ] && kill -0 "$$(cat "$(SITL_PID_FILE)")" 2>/dev/null; then \
		echo "SITL '$(NAME)' is already running with pid $$(cat "$(SITL_PID_FILE)")"; \
		exit 1; \
	fi
	@echo "Starting SITL '$(NAME)' in background. Log: $(SITL_LOG_FILE)"
	@nohup sh -c 'cd "$$1" || exit 1; exec Tools/autotest/sim_vehicle.py -v "$$2" -f "$$3" -I "$$4" --use-dir "$$5" --aircraft "$$6" $$7 --mavproxy-args="$$8" $$9' \
		sh "$(abspath $(ROOT))" "$(VEHICLE)" "$(TYPE)" "$(INSTANCE)" "$(SITL_DIR)" "$(NAME)" '$(SITL_PARAM_ARG)' '$(MAVPROXY_ARGS)' '$(SITL_EXTRA_ARGS)' \
		> "$(SITL_LOG_FILE)" 2>&1 & echo $$! > "$(SITL_PID_FILE)"
	@echo "Started pid $$(cat "$(SITL_PID_FILE)")"

sitl-run-fg: sitl-create
	@cd "$(ROOT)" && Tools/autotest/sim_vehicle.py \
		-v "$(VEHICLE)" \
		-f "$(TYPE)" \
		-I "$(INSTANCE)" \
		--use-dir "$(SITL_DIR)" \
		--aircraft "$(NAME)" \
		$(SITL_PARAM_ARG) \
		--console \
		--map \
		--mavproxy-args="$(MAVPROXY_ARGS)" \
		$(SITL_EXTRA_ARGS)

sitl-stop:
	@if [ ! -f "$(SITL_PID_FILE)" ]; then \
		echo "No pid file for SITL '$(NAME)'"; \
		exit 0; \
	fi; \
	pid=$$(cat "$(SITL_PID_FILE)"); \
	if kill -0 "$$pid" 2>/dev/null; then \
		echo "Stopping SITL '$(NAME)' pid $$pid"; \
		kill "$$pid"; \
		for _ in 1 2 3 4 5; do \
			kill -0 "$$pid" 2>/dev/null || break; \
			sleep 1; \
		done; \
		if kill -0 "$$pid" 2>/dev/null; then \
			echo "SITL '$(NAME)' did not stop after SIGTERM; sending SIGKILL"; \
			kill -KILL "$$pid"; \
		fi; \
	else \
		echo "Removing stale pid file for SITL '$(NAME)'"; \
	fi; \
	rm -f "$(SITL_PID_FILE)"

sitl-status:
	@if [ -f "$(SITL_PID_FILE)" ] && kill -0 "$$(cat "$(SITL_PID_FILE)")" 2>/dev/null; then \
		echo "SITL '$(NAME)' is running with pid $$(cat "$(SITL_PID_FILE)")"; \
		echo "Directory: $(SITL_DIR)"; \
		echo "Log: $(SITL_LOG_FILE)"; \
	else \
		echo "SITL '$(NAME)' is not running"; \
	fi

sitl-tail:
	@tail -f "$(SITL_LOG_FILE)"

sitl-wipe: sitl-stop
	@rm -f "$(SITL_DIR)/eeprom.bin" "$(SITL_DIR)/persistent.dat"
	@echo "Removed EEPROM/state for SITL '$(NAME)'"

help:
	@echo "Ardupilot Building"
	@echo "=================="
	@echo "This is a make wrapper for Ardupilot's Waf build system. This wrapper is"
	@echo "intended to provide convenience for basic and common build tasks. If you need"
	@echo "more than what this wrapper provides, it's a good idea to use waf directly."
	@echo "The waf executable is at '$(WAF_BINARY)'."
	@echo ""
	@echo "For more detailed instructions see https://ardupilot.org/dev/docs/building-the-code.html"
	@echo ""
	@echo "Boards"
	@echo "------"
	@echo ""
	@echo "In order to trigger the build for a board/platform, the name of the board is"
	@echo "used as the target. Example: make linux"
	@echo "If no target is passed, then the build will be triggered for the last board"
	@echo "used. You can suffix the board/platform with '-configure' in order to just "
	@echo "configure without triggering a build command."
	@echo ""
	@echo "You can get a list of available boards using the command:"
	@echo "    make list_boards"
	@echo ""
	@echo "Vehicles"
	@echo "--------"
	@echo ""
	@echo "It's possible to build for a specific vehicle by defining the target as one of:"
	@echo "    $(VEHICLES)"
	@echo ""
	@echo "Not that if it's your first time building or you want to change the target "
	@echo "board/platform, you'll need to configure the build before (e.g"
	@echo "make linux-configure)"
	@echo ""
	@echo "Combinations"
	@echo "------------"
	@echo ""
	@echo "It's possible to build for a specific vehicle type and board using"
	@echo "    make BOARD-VEHICLE"
	@echo ""
	@echo "For example, to build copter for the Pixracer, use this:"
	@echo "    make Pixracer-copter"
	@echo ""
	@echo "Check"
	@echo "-----"
	@echo ""
	@echo "Check targets are used for running tests. There are two targets available:"
	@echo "    check:	 for running tests that are still failing or that are new or"
	@echo "    		   have been modified"
	@echo "    check-all: to run all tests"
	@echo ""
	@echo "Waf commands"
	@echo "------------"
	@echo ""
	@echo "Waf commands can be explicitly called with targets prefixed by 'waf-'. Example:"
	@echo "    make waf-clean"
	@echo "    make waf-build"
	@echo ""
	@echo "Common commands"
	@echo "---------------"
	@echo ""
	@echo "Some Waf commands can be executed without the need of prefixing the target name"
	@echo "with 'waf-'. They are: $(EXPLICIT_COMMANDS)"
	@echo ""
	@echo "Waf flags"
	@echo "---------"
	@echo ""
	@echo "The variable WAF_FLAGS can be used to set any waf command line options that"
	@echo "come to be necessary. Ex: make linux WAF_FLAGS='-c no'"
	@echo ""
	@echo "SITL profiles"
	@echo "-------------"
	@echo ""
	@echo "Named SITL runtime profiles can be created and started without adding vehicle"
	@echo "source directories. State is stored under SITL_ROOT/NAME, defaulting to"
	@echo "'$(SITL_ROOT)/$(NAME)'."
	@echo ""
	@echo "Targets:"
	@echo "    make sitl-create NAME=cessna TYPE=plane"
	@echo "    make sitl-run NAME=cessna TYPE=plane TCP_PORT=5765 UDP_PORT=14550"
	@echo "    make sitl-run-fg NAME=cessna TYPE=plane"
	@echo "    make sitl-status NAME=cessna"
	@echo "    make sitl-stop NAME=cessna"
	@echo "    make sitl-wipe NAME=cessna"
	@echo ""
	@echo "Common variables:"
	@echo "    NAME            profile directory name"
	@echo "    TYPE            SITL frame/model, for example plane, xplane, quad, rover"
	@echo "    VEHICLE         ArduPlane, ArduCopter, Rover, or ArduSub"
	@echo "    INSTANCE        SITL instance number; use a unique value per running sim"
	@echo "    TCP_PORT        MAVSDK TCP listen port, default $(TCP_PORT)"
	@echo "    QGC_HOST        QGroundControl host, default $(QGC_HOST)"
	@echo "    UDP_PORT        QGroundControl UDP port, default $(UDP_PORT)"
	@echo "    MAVPROXY_ARGS   full MAVProxy output args override"
	@echo "    SITL_EXTRA_ARGS extra sim_vehicle.py args"
	@echo ""
	@echo "Example for two simultaneous vehicles:"
	@echo "    make sitl-run NAME=cessna TYPE=plane INSTANCE=0 TCP_PORT=5765 UDP_PORT=14550"
	@echo "    make sitl-run NAME=quad1 TYPE=quad INSTANCE=1 TCP_PORT=5775 UDP_PORT=14551"
	@echo ""

# Don't run in parallel, let waf take care of that.
.NOTPARALLEL:
