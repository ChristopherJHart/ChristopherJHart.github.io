---
layout: post
title: Refactoring Python Network Automation for Unit Tests
---

Network operators who embrace automation to streamline their daily work often adopt software development best practices to ensure the applications they develop are stable and of the highest quality. One such best practice is the implementation of unit tests to ensure the core business logic of an application behaves as expected, especially as the feature set of the application grows over time. However, determining how to [refactor](https://en.wikipedia.org/wiki/Code_refactoring) computer code to implement unit tests can be difficult for network operators new to software development.

In this post, we'll explore how Python-based network automation can be refactored to take advantage of unit tests using functions.

## Common Network Automation Design Patterns

Network automation software written to solve a specific organizational business problem tends to do three high-level things:

1. Connect to one or more network devices and retrieve data from it/them, typically via the output of a show command or API endpoint.
2. Parse and/or transform the data so that it solves the business problem. This typically involves:
    * Cleaning up or normalizing the data, such as converting short interface names (Gi1/0/1) to long interface names (GigabitEthernet1/0/1).
    * Reducing or filtering the data to a more narrow scope, such as targeting a specific type of interface.
    * Changing the format, such as converting [XML](https://en.wikipedia.org/wiki/XML) to [JSON](https://en.wikipedia.org/wiki/JSON).
3. Some form of I/O (Input/Output), such as writing relevant information to a database, displaying pertinent output to the user, or updating data in another application, such as an IPAM (IP Address Management) system, CMDB (Configuration Management Database), or ITSM (Information Technology Service Management) system.

A key advantage of the Python programming language in the network automation realm is that one can do a *lot* of work with very little code. Depending on the specific libraries used, all 3 of the above items can be accomplished in 50-100 lines of code. Since the total amount of code written is so low and often fits inside a single function, network operators may try to implement unit testing for this function.

The first and last items of the above list are difficult to unit test, since they rely on external dependencies (a network device, a database, a console to write to, etc.) to be online and operational for the unit tests to execute, which is not always possible. Furthermore, these external dependencies are often libraries or applications that have their own comprehensive suite of tests that ensure it is stable; in other words, there's no point in writing unit tests for software that you don't own or control.

However, the second item of the above list is much easier to unit test. Furthermore, it is the section of our application that is most likely to change over time as more features are added, which also means it is most likely where bugs will be located. Therefore, code that falls under the second item of the above list is where unit tests will have the most value, so that is where we will want to start.

> **Note**: High-quality, production-grade applications *should* test the first and last items of the above list! However, these tests will most likely be *integration* tests, not necessarily *unit* tests.

## Refactoring For Unit Tests with Functions

In Python, the smallest module of code that most software developers will unit test is a function. Functions are often used for [code reuse purposes](https://en.wikipedia.org/wiki/Modular_programming#History) and allow software developers to abide by the [DRY (Don't Repeat Yourself) principle of software development](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself). However, functions also create a ["contract"](https://en.wikipedia.org/wiki/Design_by_contract), allowing the inputs and outputs of a software component to be rigidly defined (and, therefore, easily tested). Therefore, if we wish to implement unit tests in our network automation, we need to break up the tasks of our application into multiple functions.

As previously mentioned, because Python is such a powerful language that can do a lot of work with very little code, all of the code in our application may fit into a single large function that:

1. Connects to a device via SSH and runs some commands.
2. Transforms the output into something usable.
3. Returns some meaningful output to the user.

This large function cannot be unit tested since it relies upon external dependencies. We can refactor this large function into multiple smaller functions, then use a fourth function (called a ["wrapper function"](https://en.wikipedia.org/wiki/Wrapper_function)) to execute each of these smaller functions in sequence.

This refactoring is best demonstrated through a practical example.

## The Business Problem

Let's say that as a network operator, you are responsible for a [large hierarchical (three-layer) enterprise campus network with core, distribution, and access layers](https://www.ciscopress.com/articles/article.asp?p=2202410&seqNum=4). A weekly task assigned to you is to analyze the MAC address table of each switch in this network to footprint how many MAC addresses are dynamically learned off of each interface of each switch. The purpose of this task is to monitor the network for large shifts in MAC addresses dynamically learned between different sections of the network with the intent of identifying anomalies that may indicate a problem.

To make matters worse, the switches in this network span multiple different vendors and multiple different platforms within each vendor. This means the command used to display the MAC address table will vary, and the output of that command will also vary.

This task is extremely laborious and error-prone when performed manually, so it is an obvious target for automation. At a high-level, our automation will take the following tasks to solve this business problem:

1. Log into one or more devices via SSH.
2. Execute the relevant command needed to display the MAC addresses learned by the switch as unstructured data.
3. Parse the unstructured data and transform it into structured data.
4. Extract relevant data from the structured data.
5. Store the relevant data in a database.

## A Monolithic Solution

As part of a proof of concept, you created the below Python application to solve this business problem. This application does the following:

1. Takes in command line arguments defining the IP address/FQDN (Fully Qualified Domain Name) of the switch to analyze using the [argparse library](https://docs.python.org/3/library/argparse.html).
2. Uses the [Scrapli library](https://github.com/carlmontanari/scrapli) to SSH into the switch and execute the `show mac address-table` command.
3. Uses the [Genie library](https://developer.cisco.com/docs/genie-docs/) to parse the output of the `show mac address-table` command from a Cisco IOS switch into structured data.
4. Counts the quantity of MAC addresses dynamically learned on each interface of the switch using the structured data from Genie. Static MAC addresses are ignored.
5. Stores the switch, interface, and MAC address count tuple in a SQLite database using the [sqlite3 library](https://docs.python.org/3/library/sqlite3.html).

```python
from argparse import ArgumentParser
import os
import sqlite3
import sys
import scrapli


parser = ArgumentParser(
    description="Gather and store per-interface MAC address count from switch."
)

parser.add_argument("--debug", help="Enable debug logging", action="store_true")
parser.add_argument(
    "host",
    help="IP address of device to gather MAC addresses from.",
    action="store",
)
parser.add_argument(
    "username",
    help="Username to use to log into device.",
    action="store",
)
parser.add_argument(
    "password",
    help="Password to use to log into device.",
    action="store",
)

args = parser.parse_args()

def main() -> None:
    # Fetch MAC address table from device as structured data.
    with scrapli.driver.core.IOSXEDriver(
        host=args.host,
        auth_username=args.username,
        auth_password=args.password,
        auth_strict_key=False,
        ssh_config_file=True,
    ) as conn:
        response = conn.send_command("show mac address-table")
    structured_data = response.genie_parse_output()

    # Extract information from structured data
    interface_mac_count_data = {}
    for vlan_id, vlan_data in structured_data["mac_table"]["vlans"].items():
        for mac_address, mac_address_data in vlan_data["mac_addresses"].items():
            for interface, interface_data in mac_address_data["interfaces"].items():
                if interface_data.get("entry_type", "") == "dynamic":
                    # This interface is interesting to us, so update interface_mac_count_data
                    # accordingly.
                    if interface_mac_count_data.get(interface) is not None:
                        interface_mac_count_data[interface] += 1
                    else:
                        interface_mac_count_data[interface] = 1

    # Create SQLite database with information. If the file already exists, delete it first.
    if os.path.exists("network_mac_count.db"):
        os.remove("network_mac_count.db")
    db_connection = sqlite3.connect("network_mac_count.db")
    db_cursor = db_connection.cursor()
    # Create database table if it doesn't already exist.
    db_cursor.execute(
        """
        CREATE TABLE IF NOT EXISTS mac_count (
            switch_name CHAR,
            interface_name CHAR,
            mac_count INT
        );
        """
    )
    for interface, mac_count in interface_mac_count_data.items():
        db_cursor.execute(
            f"""
            INSERT INTO mac_count VALUES (
                '{args.host}',
                '{interface}',
                {mac_count}
            );
            """
        )
    db_connection.commit()

    # Read back information from the database and print to the console.
    for row in db_cursor.execute("SELECT * FROM mac_count;"):
        print(row)

    db_connection.close()


if __name__ == "__main__":
    try:
        main()
    except KeyboardInterrupt:
        sys.exit()
```

When this program is executed against a switch in my homelab, the following output is displayed.

```
(venv) christopher@ubuntu-playground:~/Documents/Python/network-automation-testing$ python monolith_automation.py 192.168.10.3 automation automation
('192.168.10.3', 'GigabitEthernet0/14', 1)
('192.168.10.3', 'GigabitEthernet0/2', 14)
('192.168.10.3', 'GigabitEthernet0/3', 5)
('192.168.10.3', 'GigabitEthernet0/6', 1)
('192.168.10.3', 'GigabitEthernet0/13', 2)
('192.168.10.3', 'GigabitEthernet0/5', 1)
('192.168.10.3', 'GigabitEthernet0/4', 1)
```

As you can see, this application works as designed, and is very efficient - a sizeable chunk of our daily work has been solved in just under 60 lines of Python code! However, this automation only works against Cisco IOS-XE switches, and our network consists of a diverse multi-vendor platform. As we add support for more platforms, we will undoubtedly need to modify the section of our code that extracts interface MAC address counts *without* breaking our existing support for Cisco IOS-XE switches. To do this safely, we will need to add unit tests, which means we need to refactor this software.

## A Refactored, Modular Solution

Our current application has a single function, `main()`, that executes the application. We can break the core of this application up into four separate functions:

1. `fetch_structured_data_from_device` connects to the switch via SSH using Scrapli and returns the structured data of the switch's MAC address table.
2. `extract_interface_mac_address_count` extracts relevant data from the structured data of the switch's MAC address table and transforms it into a format we want.
3. `update_database` pushes the data we're interested in into a database for long-term storage.
4. `main` acts as our wrapper function that executes `fetch_structured_data_from_device`, `extract_interface_mac_address_count`, and `update_database` in sequence.

The parsing of arguments using the argparse library is also refactored into its own function, which is called within the `main` function. This ensures the library we use for unit testing can import individual functions from this application without accidentally invoking the argument parsing code.

The refactored application is shown below.

```python
from argparse import ArgumentParser, Namespace
import os
from pprint import pprint
import sqlite3
import sys
from typing import Dict
import scrapli


def parse_args() -> Namespace:
    """Parse command line arguments."""
    parser = ArgumentParser(description="Fetches MAC addresses from switch.")

    parser.add_argument("--debug", help="Enable debug logging", action="store_true")
    parser.add_argument(
        "host",
        help="IP address of device to gather MAC addresses from.",
        action="store",
    )
    parser.add_argument(
        "username",
        help="Username to use to log into device.",
        action="store",
    )
    parser.add_argument(
        "password",
        help="Password to use to log into device.",
        action="store",
    )

    return parser.parse_args()


def fetch_structured_data_from_device(host: str, username: str, password: str) -> Dict:
    """Fetch MAC address table from switch and return as structured data."""
    with scrapli.driver.core.IOSXEDriver(
        host=host,
        auth_username=username,
        auth_password=password,
        auth_strict_key=False,
        ssh_config_file=True,
    ) as conn:
        response = conn.send_command("show mac address-table")
    return response.genie_parse_output()


def extract_interface_mac_address_count(data: Dict) -> Dict[str, int]:
    """Extract and return relevant data from MAC address table structured data."""
    interface_mac_count_data = {}
    for vlan_id, vlan_data in data["mac_table"]["vlans"].items():
        for mac_address, mac_address_data in vlan_data["mac_addresses"].items():
            for interface, interface_data in mac_address_data["interfaces"].items():
                if interface_data.get("entry_type", "") == "dynamic":
                    # This interface is interesting to us, so update interface_mac_count_data
                    # accordingly.
                    if interface_mac_count_data.get(interface) is not None:
                        interface_mac_count_data[interface] += 1
                    else:
                        interface_mac_count_data[interface] = 1
    return interface_mac_count_data


def update_database(host: str, interface_data: Dict[str, int]) -> None:
    """Create and update SQLite database with per-interface MAC address count."""
    # Create SQLite database with information. If the file already exists, delete it first.
    if os.path.exists("network_mac_count.db"):
        os.remove("network_mac_count.db")
    db_connection = sqlite3.connect("network_mac_count.db")
    db_cursor = db_connection.cursor()

    # Create database table if it doesn't already exist.
    db_cursor.execute(
        """
        CREATE TABLE IF NOT EXISTS mac_count (
            switch_name CHAR,
            interface_name CHAR,
            mac_count INT
        );
        """
    )
    for interface, mac_count in interface_data.items():
        db_cursor.execute(
            f"""
            INSERT INTO mac_count VALUES (
                '{host}',
                '{interface}',
                {mac_count}
            );
            """
        )
    db_connection.commit()

    # Read back information from the database and print to the console.
    for row in db_cursor.execute("SELECT * FROM mac_count;"):
        print(row)

    db_connection.close()


def main() -> None:
    # Parse arguments
    args = parse_args()

    # Fetch MAC address table from device as structured data.
    structured_data = fetch_structured_data_from_device(
        args.host, args.username, args.password
    )

    # Extract information from structured data
    interface_mac_count_data = extract_interface_mac_address_count(structured_data)

    # Update database with switch, interface, and MAC address count data.
    update_database(args.host, interface_mac_count_data)


if __name__ == "__main__":
    try:
        main()
    except KeyboardInterrupt:
        sys.exit()
```

If we execute this refactored program against a switch in my homelab, the following output is displayed. This output is identical to the previous version of the application, which means our refactoring did not appear to break any of our logic.

```
(venv) christopher@ubuntu-playground:~/Documents/Python/network-automation-testing$ python refactored_automation.py 192.168.10.3 automation automation
('192.168.10.3', 'GigabitEthernet0/14', 1)
('192.168.10.3', 'GigabitEthernet0/2', 14)
('192.168.10.3', 'GigabitEthernet0/3', 5)
('192.168.10.3', 'GigabitEthernet0/6', 1)
('192.168.10.3', 'GigabitEthernet0/13', 2)
('192.168.10.3', 'GigabitEthernet0/5', 1)
('192.168.10.3', 'GigabitEthernet0/4', 1)
```

## Writing Unit Tests for the Modular Solution

Now that the application is refactored, we can write unit tests for the `extract_interface_mac_address_count` function. This function is our core business logic that does not have any external dependencies, which makes it easy to write unit tests using [pytest](https://docs.pytest.org/en/7.1.x/). The below unit tests use the [`@pytest.mark.parametrize` decorator](https://docs.pytest.org/en/7.1.x/how-to/parametrize.html#parametrize-basics) to easily define the inputs and outputs of the `extract_interface_mac_address_count` function. This code contains three unit tests:

1. The input structured data contains a single MAC address learned on a single interface in a single VLAN.
2. The input structured data contains two MAC addresses learned on two separate interfaces in a single VLAN.
3. The input structured data contains two MAC addresses learned on two separate interfaces in two separate VLANs.

```python
from typing import Dict

import pytest

from refactored_automation import extract_interface_mac_address_count


@pytest.mark.parametrize(
    ["input_data", "expected_data"],
    [
        pytest.param(
            {
                "mac_table": {
                    "vlans": {
                        "1": {
                            "mac_addresses": {
                                "0000.1111.2222": {
                                    "interfaces": {
                                        "GigabitEthernet0/1": {
                                            "entry_type": "dynamic",
                                            "interface": "GigabitEthernet0/1",
                                        }
                                    },
                                    "mac_address": "0000.1111.2222",
                                },
                            },
                            "vlan": 1,
                        },
                    }
                }
            },
            {"GigabitEthernet0/1": 1},
        ),
        pytest.param(
            {
                "mac_table": {
                    "vlans": {
                        "1": {
                            "mac_addresses": {
                                "0000.1111.2222": {
                                    "interfaces": {
                                        "GigabitEthernet0/1": {
                                            "entry_type": "dynamic",
                                            "interface": "GigabitEthernet0/1",
                                        }
                                    },
                                    "mac_address": "0000.1111.2222",
                                },
                                "0000.3333.4444": {
                                    "interfaces": {
                                        "GigabitEthernet0/2": {
                                            "entry_type": "dynamic",
                                            "interface": "GigabitEthernet0/2",
                                        }
                                    },
                                    "mac_address": "0000.3333.4444",
                                },
                            },
                            "vlan": 1,
                        },
                    }
                }
            },
            {"GigabitEthernet0/1": 1, "GigabitEthernet0/2": 1},
        ),
        pytest.param(
            {
                "mac_table": {
                    "vlans": {
                        "1": {
                            "mac_addresses": {
                                "0000.1111.2222": {
                                    "interfaces": {
                                        "GigabitEthernet0/1": {
                                            "entry_type": "dynamic",
                                            "interface": "GigabitEthernet0/1",
                                        }
                                    },
                                    "mac_address": "0000.1111.2222",
                                },
                            },
                            "vlan": 1,
                        },
                        "2": {
                            "mac_addresses": {
                                "0000.3333.4444": {
                                    "interfaces": {
                                        "GigabitEthernet0/2": {
                                            "entry_type": "dynamic",
                                            "interface": "GigabitEthernet0/2",
                                        }
                                    },
                                    "mac_address": "0000.3333.4444",
                                },
                            },
                            "vlan": 2,
                        },
                    }
                }
            },
            {"GigabitEthernet0/1": 1, "GigabitEthernet0/2": 1},
        ),
    ],
)
def test_extract_interface_mac_address_count(
    input_data: Dict, expected_data: Dict[str, int]
) -> None:
    assert extract_interface_mac_address_count(input_data) == expected_data
```

If we execute these unit tests with the `pytest` command, we can see that all tests are passing. Hooray!

```
(venv) christopher@ubuntu-playground:~/Documents/Python/network-automation-testing$ pytest
======================================================================= test session starts =======================================================================
platform linux -- Python 3.10.4, pytest-7.1.3, pluggy-1.0.0
rootdir: /home/christopher/Documents/Python/network-automation-testing
collected 3 items

test_refactored_automation.py ...                                                                                                                           [100%]

======================================================================== 3 passed in 0.11s ========================================================================
```

## Conclusion

From here, we can add additional tests that validate different possible permutations of the structured data fed into the `extract_interface_mac_address_count` function. This allows us to harden our code to handle multiple scenarios, resulting in higher quality code. Furthermore, we can make changes to this function in the future and have a high degree of confidence that the changes did not break previously-working functionality. Finally, when defects are discovered with our code, we can fix them *and* add them to the unit tests for the function to ensure [software regressions](https://en.wikipedia.org/wiki/Software_regression) do not appear in our code over time.

For example, right now, the `extract_interface_mac_address_count` function would break if an empty dictionary was passed into the function through the `data` parameter. This is a theroetically possible value - after all, what if a switch does not have any dynamically-learned MAC addresses present in its MAC address table? After fixing this bug, we could add a unit test that passes an empty dictionary into the `extract_interface_mac_address_count` function to ensure that the same bug does not reappear at some point in the future.

In conclusion, network operators developing network automation should try to refactor their applications such that core business logic not reliant on external dependencies is contractually defined through functions. This allows for unit tests to ensure the core business logic is functioning as expected, which is important as the application's feature set grows over time.

## Acknowledgments

I'd like to thank [Ken Celenza](https://twitter.com/itdependsnet) for encouraging me to put this information into a blog post. I'd also like to plug the [Network To Code Slack channel](http://slack.networktocode.com/), which is where the core of this discussion originated!
