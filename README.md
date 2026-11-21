[![PyPI Version][pypi-image]][pypi-url]
[![Downloads](https://static.pepy.tech/badge/aiopvpc-ng)](https://pepy.tech/projects/aiopvpc-ng)
[![Build Status][build-image]][build-url]
[![License][license-image]][license-url]

<!-- Badges -->

[pypi-image]: https://img.shields.io/pypi/v/aiopvpc-ng
[pypi-url]: https://pypi.org/project/aiopvpc-ng/
[build-image]: https://github.com/marcomsousa/aiopvpc-ng/actions/workflows/main.yml/badge.svg
[build-url]: https://github.com/marcomsousa/aiopvpc-ng/actions/workflows/main.yml
[license-image]: https://img.shields.io/github/license/marcomsousa/aiopvpc-ng
[license-url]: https://github.com/marcomsousa/aiopvpc-ng/blob/master/LICENSE

# aiopvpc-ng

Next-generation async Python library (`>=3.11`) to download Spanish electricity prices (PVPC 2.0TD, surplus/injection, MAG, OMIE, market adjustment, and indexed tariffs) from Red Eléctrica de España's ESIOS API (`api.esios.ree.es`).

`aiopvpc-ng` is an actively maintained, drop-in successor to [`azogue/aiopvpc`](https://github.com/azogue/aiopvpc), keeping the `aiopvpc` import namespace for full compatibility with the [**`pvpc_hourly_pricing`** Home Assistant integration](https://www.home-assistant.io/integrations/pvpc_hourly_pricing/) and standalone Python tools.

## Install

Requires **Python 3.11 or newer**.

Install with `pip install aiopvpc-ng` (or `uv add aiopvpc-ng`).

For local development and testing with [`uv`](https://docs.astral.sh/uv/):

```bash
uv sync --locked
uv run pytest
```

## Usage

```python
from datetime import UTC, datetime
import aiohttp
from aiopvpc import PVPCData

async with aiohttp.ClientSession() as session:
    pvpc_handler = PVPCData(session=session, tariff="2.0TD")
    esios_data = await pvpc_handler.async_update_all(
        current_data=None, now=datetime.now(UTC)
    )
print(esios_data.sensors["PVPC"])
```

## Attribution

Originally created by Eugenio Panadero ([`azogue/aiopvpc`](https://github.com/azogue/aiopvpc)). Maintained as `aiopvpc-ng` under the MIT License.
