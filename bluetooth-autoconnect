#!/usr/bin/env python3

import getopt
import os
import signal
import sys

import dbus

from dbus.mainloop.glib import DBusGMainLoop
from functools import partial
from gi.repository import GLib
from xml.etree import ElementTree


SCRIPT_NAME = os.path.basename(sys.argv[0])


def dbus_get_child_object_paths(object_path):
    object_paths = []
    obj = bus.get_object('org.bluez', object_path, introspect=False)
    xml_string = obj.Introspect(dbus_interface='org.freedesktop.DBus.Introspectable')

    if object_path == '/':
        object_path = ''

    for child in ElementTree.fromstring(xml_string):
        if child.tag == 'node':
            object_paths.append(object_path + '/' + child.attrib['name'])

    return object_paths


def connect_devices_for_adapter(adapter_path):
    if verbose:
        print(f'scanning for devices on {adapter_path}', flush=True)

    # Get list of devices
    device_paths = dbus_get_child_object_paths(adapter_path)

    # Manage pending connections for non-daemon mode
    def add_pending_connection(device_path):
        if not daemon:
            pending_connections.add(device_path)

    def remove_pending_connection(device_path):
        if not daemon:
            pending_connections.discard(device_path)
            # If this was the last pending connection, terminate the
            # mainloop
            if len(pending_connections) == 0:
                loop.quit()

    # Handle the async replies for a connection attempt
    def reply_handler(device_path):
        print(f'successfully connected to device {device_path}', flush=True)
        remove_pending_connection(device_path)

    def error_handler(device_path, e):
        print(f'error connecting to device {device_path}: {e.get_dbus_message()}', file=sys.stderr, flush=True)
        remove_pending_connection(device_path)

    for device_path in device_paths:
        if verbose:
            print(f'found device {device_path}', flush=True)
        try:
            # Read the device's properties
            obj = bus.get_object('org.bluez', device_path)
            props = obj.GetAll(
                    'org.bluez.Device1',
                    dbus_interface='org.freedesktop.DBus.Properties')

            if props.get('Connected', False):
                if verbose:
                    print(f'device {device_path} is already connected', flush=True)
            elif not props.get('Trusted', False):
                if verbose:
                    print(f'device {device_path} is not trusted', flush=True)
            else:
                print(f'connecting to device {device_path}', flush=True)
                add_pending_connection(device_path)

                # Attempt to connect to the device
                obj.Connect(
                        dbus_interface='org.bluez.Device1',
                        reply_handler=partial(reply_handler, device_path),
                        error_handler=partial(error_handler, device_path))
        except dbus.exceptions.DBusException as e:
            error_handler(device_path, e)


def connect_devices_for_all_adapters():
    if verbose:
        print('scanning for adapters...', flush=True)

    # Get a list of adapters
    adapter_paths = dbus_get_child_object_paths('/org/bluez')

    for adapter_path in adapter_paths:
        if verbose:
            print(f'found adapter {adapter_path}', flush=True)
        try:
            # Read the adapter's properties
            obj = bus.get_object('org.bluez', adapter_path)
            props = obj.GetAll(
                    'org.bluez.Adapter1',
                    dbus_interface='org.freedesktop.DBus.Properties')

            # We're only interested in adapters that are powered on
            if props.get('Powered', False):
                if verbose:
                    print(f'adapter {adapter_path} is powered on', flush=True)

                # Try to connect to devices on this adapter
                connect_devices_for_adapter(adapter_path)
        except dbus.exceptions.DBusException as e:
            print(f'error reading properties for adapter {adapter_path}: {e.get_dbus_message()}', file=sys.stderr, flush=True)


def properties_changed_handler(interface_name, changed_properties, invalidated_properties, path):
    # We're only interested in adapters that have been powered on
    if interface_name == 'org.bluez.Adapter1' and changed_properties.get('Powered', False):
        if verbose:
            print(f'adapter {path} has powered on', flush=True)

        # Try to connect to devices on this adapter
        connect_devices_for_adapter(path)


def usage():
    print('\n'.join([
        f'Usage: {SCRIPT_NAME} [OPTIONS]...',
        f'',
        f'Automatically connect to trusted bluetooth devices',
        f'',
        f'OPTIONS:',
        f'  -d, --daemon      Monitor bluetooth adapters and automatically connect to',
        f'                    trusted devices when an adapter is powered on',
        f'  -h, --help        Print this help message',
        f'  -v, --verbose     Show more detailed log messages',
        f'',
    ]), flush=True)
    sys.exit(0)


def main():
    global daemon
    global verbose

    # Parse command line arguments
    try:
        opts, cmds = getopt.getopt(sys.argv[1:], 'dhv', ['daemon', 'help', 'verbose'])
    except getopt.GetoptError as e:
        print(f'{SCRIPT_NAME}:', e, file=sys.stderr, flush=True)
        print(f"Try '{SCRIPT_NAME} --help' for more information", file=sys.stderr, flush=True)
        sys.exit(1)

    # Process options (e.g. -h, --verbose)
    for o, v in opts:
        if o in ('-d', '--daemon'):
            daemon = True
        elif o in ('-h', '--help'):
            usage()
        elif o in ('-v', '--verbose'):
            verbose = True
        else:
            # This shouldn't ever happen unless we forget to handle an
            # option we've added
            print(f'{SCRIPT_NAME}: internal error: unhandled option {o}', file=sys.stderr, flush=True)
            sys.exit(1)

    # Process commands
    # This script does not use any commands so we will exit if one is
    # incorrectly provided
    if len(cmds) > 0:
        print(f"{SCRIPT_NAME}: command '{c}' not recognized", file=sys.stderr, flush=True)
        print(f"Try '{SCRIPT_NAME} --help' for more information", file=sys.stderr, flush=True)
        sys.exit(1)

    # Set process name and title
    # This allows commands like `killall SCRIPT_NAME` to function
    # correctly
    try:
        import prctl
        if verbose:
            print(f'setting process name to \'{SCRIPT_NAME}\'', flush=True)
        prctl.set_name(SCRIPT_NAME)
        prctl.set_proctitle(' '.join(sys.argv))
    except ImportError:
        if verbose:
            print(f'failed to load module \'prctl\'', flush=True)
            print(f'process name not set', flush=True)

    if daemon:
        # Listen for changes on the BlueZ dbus interface
        # This is a catch all listener (no path specified) because we
        # want to get notified for all adapters without keeping a list
        # of them and managing signal handlers independantly
        bus.add_signal_receiver(
                properties_changed_handler,
                signal_name='PropertiesChanged',
                dbus_interface='org.freedesktop.DBus.Properties',
                bus_name='org.bluez',
                path=None,
                path_keyword='path')

        # Attempt to connect to devices on all existing adapters
        connect_devices_for_all_adapters()

        # Start the mainloop
        loop.run()
    else:
        # Attempt to connect to devices on all existing adapters
        connect_devices_for_all_adapters()

        # If we're waiting for connection attemps to finish, start the
        # mainloop. We will automatically exit the loop once everything
        # is finished
        if len(pending_connections) > 0:
            loop.run()


def signal_handler(sig, frame):
    if sig == signal.SIGHUP:
        # Rescan adapters and attempt to connect to devices if we're in
        # daemon mode
        if daemon:
            connect_devices_for_all_adapters()
    elif sig in (signal.SIGINT, signal.SIGTERM):
        # Gracefully exit
        sys.exit(0)
    else:
        # This shouldn't ever happen unless we forget to handle a signal
        # we've added
        print(f'internal error: unhandled signal {sig}', file=sys.stderr, flush=True)
        sys.exit(2)


if __name__ == '__main__':
    # Register signal handlers
    signal.signal(signal.SIGHUP, signal_handler)
    signal.signal(signal.SIGINT, signal_handler)
    signal.signal(signal.SIGUSR1, signal.SIG_IGN)
    signal.signal(signal.SIGUSR2, signal.SIG_IGN)
    signal.signal(signal.SIGALRM, signal.SIG_IGN)
    signal.signal(signal.SIGTERM, signal_handler)

    # Connect to the system dbus
    DBusGMainLoop(set_as_default=True)
    bus = dbus.SystemBus()

    # Initialize globals
    daemon = False
    verbose = False
    pending_connections = set()

    # Initialize the mainloop, but don't start it yet
    loop = GLib.MainLoop()

    main()


# vim: ft=python ts=8 et sw=4 sts=4
