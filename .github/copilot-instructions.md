# Copilot instructions for `pysim`

## Build, test, and lint commands

Use the commands already encoded in `contrib/jenkins.sh` and the test README:

```sh
# install runtime/test dependencies
pip install -r requirements.txt

# unit tests
python3 -m unittest discover -v -s tests/unittests

# run one unittest module
python3 -m unittest -v tests.unittests.test_apdu

# run one unittest test case / method
python3 -m unittest -v tests.unittests.test_apdu.TestApdu.test_successful

# pySim-shell integration tests (requires configured physical cards + PC/SC readers)
python3 -m unittest discover -v -s tests/pySim-shell_test

# run one pySim-shell integration test by name filter
python3 -m unittest discover -v -s tests/pySim-shell_test -k export_fs

# pySim-prog integration test (requires physical cards)
(cd tests/pySim-prog_test && ./pySim-prog_test.sh)

# pySim-trace test
tests/pySim-trace_test/pySim-trace_test.sh

# pySim-smpp2sim test
tests/pySim-smpp2sim_test/pySim-smpp2sim_test.sh

# lint
python3 -m pylint -j0 --errors-only \
  --disable E1102 \
  --disable E0401 \
  --enable W0301 \
  pySim tests/unittests/*.py *.py contrib/*.py

# docs
make -C docs html latexpdf
```

## High-level architecture

- The top-level scripts are thin entry points around the `pySim` package: `pySim-shell.py` is the main interactive tool, `pySim-prog.py` / `pySim-read.py` are legacy batch tools, `pySim-trace.py` decodes SIM protocol traces, `pySim-smpp2sim.py` handles SMPP/SIM workflows, and `osmo-smdpp.py` is a proof-of-concept SM-DP+ server built on the `pySim.esim` stack.
- The core runtime path is **transport -> APDU command layer -> card/profile detection -> runtime filesystem model -> cmd2 shell commands**:
  - `pySim.transport` provides `LinkBase` implementations for talking to a card and APDU tracing/proactive handling hooks.
  - `pySim.commands.SimCardCommands` wraps the transport, applies logical-channel CLA handling, and exposes APDU helpers.
  - `pySim.app.init_card()` waits for a card, detects a generic card type, picks the most specific `CardProfile`, attaches all discovered `CardApplication` subclasses for UICC cards, creates `RuntimeState`, applies matching `CardModel` overlays, and populates identities like ATR/EID.
  - `pySim.runtime.RuntimeState` builds the in-memory `MF` tree from `profile.files_in_mf`, probes optional profile add-ons, matches applications first from `EF.DIR` and then by active probing, and keeps the live session state.
  - `pySim.runtime.RuntimeLchan` is the per-logical-channel view: each lchan forks its own `SimCardCommands` instance from the shared transport and tracks selected file / selected ADF / decoded FCP state.
  - `pySim-shell.py` creates `PysimApp`, then `equip()` registers profile-level and file-level `cmd2.CommandSet`s so commands appear/disappear with the selected card/file.
- `pySim.filesystem` models the **specification** of the card filesystem, not live card contents. Concrete TS/GSMA modules such as `ts_102_221.py`, `ts_51_011.py`, `ts_31_102.py`, `ts_31_103.py`, `global_platform/`, and `euicc.py` define concrete `MF`/`DF`/`ADF`/`EF` classes, status words, and application/profile objects that are assembled into the runtime tree.
- There is a deliberate layering between generic and card-specific behavior:
  - `profile.py` decides which high-level profile matches a card and defines profile-wide status words / shell commands / default CLA handling.
  - `cards.py` handles generic SIM vs UICC behavior such as AID discovery and ADF selection.
  - `filesystem.py` and the spec modules define the tree shape plus encode/decode logic for each file type.
  - `CardModel.apply_matching_models()` is where vendor- or product-specific overlays add extra files on top of a matched generic profile.
- eUICC and GlobalPlatform support are integrated into the same model, not bolted on separately: `pySim.app` imports those modules up front so their `CardProfile` / `CardApplication` / `CardModel` subclasses are registered before detection runs.

## Key conventions

- New filesystem content is usually added by subclassing `TransparentEF`, `LinFixedEF`, `BerTlvEF`, `CardDF`, or `CardADF` in the relevant spec module. File classes typically define a `construct`-based codec (`self._construct`) plus `_test_decode`, `_test_encode`, or `_test_de_encode` fixtures so the generic tests in `tests/unittests/test_files.py` exercise them automatically.
- File-, application-, and profile-specific shell commands are attached to the model objects themselves:
  - file classes append nested `CommandSet`s to `self.shell_commands`
  - profile classes pass `shell_cmdsets=[...]`
  - `RuntimeLchan.register_cmds()` / `unregister_cmds()` wire those commands into `pySim-shell` when selection changes
  Avoid central command registries or switch statements when extending shell behavior.
- Detection priority is controlled by `CardProfile.ORDER`; lower numbers win. eUICC profiles intentionally sort ahead of generic UICC/SIM profiles.
- `CardProfile.pick()` and `CardModel.apply_matching_models()` rely on subclass discovery and import side effects. If you add a new profile/application/model family, make sure it is imported from startup code so the subclass exists before detection.
- After runtime initialization, prefer `RuntimeLchan` selection helpers over direct raw `card._scc.select_*` calls. Direct card access is only used during early initialization before the runtime file model is fully established.
- The `pySim-shell` integration suite in `tests/pySim-shell_test/` is built on `unittest`, runs against real cards/readers, and uses `config.yaml` plus `card_data.csv` to map physical cards to expected ICCID/EID/ADM data. `.ok` files are golden outputs and can be regenerated through the test config.
- `tests/unittests/test_fs_coverage.py` enforces that every filesystem-bearing `CardProfile` / `CardApplication` / standalone `CardDF` is represented in `docs/pysim_fs_sphinx.py` or explicitly excluded. When adding new filesystem structures, update the docs mapping as part of the same change.
