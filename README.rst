********
pyiCloud
********

.. image:: https://travis-ci.org/picklepete/pyicloud.svg?branch=master
    :alt: Check out our test status at https://travis-ci.org/picklepete/pyicloud
    :target: https://travis-ci.org/picklepete/pyicloud

.. image:: https://img.shields.io/pypi/v/pyicloud.svg
    :alt: Library version
    :target: https://pypi.org/project/pyicloud

.. image:: https://img.shields.io/pypi/pyversions/pyicloud.svg
    :alt: Supported versions
    :target: https://pypi.org/project/pyicloud

.. image:: https://pepy.tech/badge/pyicloud
    :alt: Downloads
    :target: https://pypi.org/project/pyicloud

.. image:: https://requires.io/github/Quentame/pyicloud/requirements.svg?branch=master
    :alt: Requirements Status
    :target: https://requires.io/github/Quentame/pyicloud/requirements/?branch=master

.. image:: https://img.shields.io/badge/code%20style-black-000000.svg
    :alt: Formated with Black
    :target: https://github.com/psf/black

.. image:: https://badges.gitter.im/Join%20Chat.svg
    :alt: Join the chat at https://gitter.im/picklepete/pyicloud
    :target: https://gitter.im/picklepete/pyicloud?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge

PyiCloud is a module which allows pythonistas to interact with iCloud webservices. It's powered by the fantastic `requests <https://github.com/kennethreitz/requests>`_ HTTP library.

At its core, PyiCloud connects to iCloud using your username and password, then performs calendar and iPhone queries against their API.


Authentication
==============

Authentication without using a saved password is as simple as passing your username and password to the ``PyiCloudService`` class:

.. code-block:: python

    from pyicloud import PyiCloudService
    api = PyiCloudService('jappleseed@apple.com', 'password')

In the event that the username/password combination is invalid, a ``PyiCloudFailedLoginException`` exception is thrown.

You can also store your password in the system keyring using the command-line tool:

.. code-block:: console

    $ icloud --username=jappleseed@apple.com
    ICloud Password for jappleseed@apple.com:
    Save password in keyring? (y/N)

If you have stored a password in the keyring, you will not be required to provide a password when interacting with the command-line tool or instantiating the ``PyiCloudService`` class for the username you stored the password for.

.. code-block:: python

    api = PyiCloudService('jappleseed@apple.com')

If you would like to delete a password stored in your system keyring, you can clear a stored password using the ``--delete-from-keyring`` command-line option:

.. code-block:: console

    $ icloud --username=jappleseed@apple.com --delete-from-keyring

**Note**: Authentication will expire after an interval set by Apple, at which point you will have to re-authenticate. This interval is currently two months.

Two-step and two-factor authentication (2SA/2FA)
************************************************

If you have enabled two-factor authentications (2FA) or `two-step authentication (2SA) <https://support.apple.com/en-us/HT204152>`_ for the account you will have to do some extra work:

.. code-block:: python

    if api.requires_2fa:
        print("Two-factor authentication required.")
        code = input("Enter the code you received of one of your approved devices: ")
        result = api.validate_2fa_code(code)
        print("Code validation result: %s" % result)

        if not result:
            print("Failed to verify security code")
            sys.exit(1)

        if not api.is_trusted_session:
            print("Session is not trusted. Requesting trust...")
            result = api.trust_session()
            print("Session trust result %s" % result)

            if not result:
                print("Failed to request trust. You will likely be prompted for the code again in the coming weeks")
    elif api.requires_2sa:
        import click
        print("Two-step authentication required. Your trusted devices are:")

        devices = api.trusted_devices
        for i, device in enumerate(devices):
            print(
                "  %s: %s" % (i, device.get('deviceName',
                "SMS to %s" % device.get('phoneNumber')))
            )

        device = click.prompt('Which device would you like to use?', default=0)
        device = devices[device]
        if not api.send_verification_code(device):
            print("Failed to send verification code")
            sys.exit(1)

        code = click.prompt('Please enter validation code')
        if not api.validate_verification_code(device, code):
            print("Failed to verify verification code")
            sys.exit(1)

Devices
=======

You can list which devices associated with your account by using the ``devices`` property:

.. code-block:: pycon

    >>> api.devices
    {
    'i9vbKRGIcLYqJnXMd1b257kUWnoyEBcEh6yM+IfmiMLh7BmOpALS+w==': <AppleDevice(iPhone 4S: Johnny Appleseed's iPhone)>,
    'reGYDh9XwqNWTGIhNBuEwP1ds0F/Lg5t/fxNbI4V939hhXawByErk+HYVNSUzmWV': <AppleDevice(MacBook Air 11": Johnny Appleseed's MacBook Air)>
    }

and you can access individual devices by either their index, or their ID:

.. code-block:: pycon

    >>> api.devices[0]
    <AppleDevice(iPhone 4S: Johnny Appleseed's iPhone)>
    >>> api.devices['i9vbKRGIcLYqJnXMd1b257kUWnoyEBcEh6yM+IfmiMLh7BmOpALS+w==']
    <AppleDevice(iPhone 4S: Johnny Appleseed's iPhone)>

or, as a shorthand if you have only one associated apple device, you can simply use the ``iphone`` property to access the first device associated with your account:

.. code-block:: pycon

    >>> api.iphone
    <AppleDevice(iPhone 4S: Johnny Appleseed's iPhone)>

Note: the first device associated with your account may not necessarily be your iPhone.

Find My iPhone
==============

Once you have successfully authenticated, you can start querying your data!

Location
********

Returns the device's last known location. The Find My iPhone app must have been installed and initialized.

.. code-block:: pycon

    >>> api.iphone.location()
    {'timeStamp': 1357753796553, 'locationFinished': True, 'longitude': -0.14189, 'positionType': 'GPS', 'locationType': None, 'latitude': 51.501364, 'isOld': False, 'horizontalAccuracy': 5.0}

Status
******

The Find My iPhone response is quite bloated, so for simplicity's sake this method will return a subset of the properties.

.. code-block:: pycon

    >>> api.iphone.status()
    {'deviceDisplayName': 'iPhone 5', 'deviceStatus': '200', 'batteryLevel': 0.6166913, 'name': "Peter's iPhone"}

If you wish to request further properties, you may do so by passing in a list of property names.

Play Sound
**********

Sends a request to the device to play a sound, if you wish pass a custom message you can do so by changing the subject arg.

.. code-block:: python

    api.iphone.play_sound()

A few moments later, the device will play a ringtone, display the default notification ("Find My iPhone Alert") and a confirmation email will be sent to you.

Lost Mode
*********

Lost mode is slightly different to the "Play Sound" functionality in that it allows the person who picks up the phone to call a specific phone number *without having to enter the passcode*. Just like "Play Sound" you may pass a custom message which the device will display, if it's not overridden the custom message of "This iPhone has been lost. Please call me." is used.

.. code-block:: python

    phone_number = '555-373-383'
    message = 'Thief! Return my phone immediately.'
    api.iphone.lost_device(phone_number, message)


Calendar
========

The calendar webservice currently only supports fetching events.

Events
******

Returns this month's events:

.. code-block:: python

    api.calendar.events()

Or, between a specific date range:

.. code-block:: python

    from_dt = datetime(2012, 1, 1)
    to_dt = datetime(2012, 1, 31)
    api.calendar.events(from_dt, to_dt)

Alternatively, you may fetch a single event's details, like so:

.. code-block:: python

    api.calendar.get_event_detail('CALENDAR', 'EVENT_ID')


Contacts
========

You can access your iCloud contacts/address book through the ``contacts`` property:

.. code-block:: pycon

    >>> for c in api.contacts.all():
    >>> print(c.get('firstName'), c.get('phones'))
    John [{'field': '+1 555-55-5555-5', 'label': 'MOBILE'}]

Note: These contacts do not include contacts federated from e.g. Facebook, only the ones stored in iCloud.


File Storage (Ubiquity)
=======================

You can access documents stored in your iCloud account by using the ``files`` property's ``dir`` method:

.. code-block:: pycon

    >>> api.files.dir()
    ['.do-not-delete',
     '.localized',
     'com~apple~Notes',
     'com~apple~Preview',
     'com~apple~mail',
     'com~apple~shoebox',
     'com~apple~system~spotlight'
    ]

You can access children and their children's children using the filename as an index:

.. code-block:: pycon

    >>> api.files['com~apple~Notes']
    <Folder: 'com~apple~Notes'>
    >>> api.files['com~apple~Notes'].type
    'folder'
    >>> api.files['com~apple~Notes'].dir()
    ['Documents']
    >>> api.files['com~apple~Notes']['Documents'].dir()
    ['Some Document']
    >>> api.files['com~apple~Notes']['Documents']['Some Document'].name
    'Some Document'
    >>> api.files['com~apple~Notes']['Documents']['Some Document'].modified
    datetime.datetime(2012, 9, 13, 2, 26, 17)
    >>> api.files['com~apple~Notes']['Documents']['Some Document'].size
    1308134
    >>> api.files['com~apple~Notes']['Documents']['Some Document'].type
    'file'

And when you have a file that you'd like to download, the ``open`` method will return a response object from which you can read the ``content``.

.. code-block:: pycon

    >>> api.files['com~apple~Notes']['Documents']['Some Document'].open().content
    'Hello, these are the file contents'

Note: the object returned from the above ``open`` method is a `response object <http://www.python-requests.org/en/latest/api/#classes>`_ and the ``open`` method can accept any parameters you might normally use in a request using `requests <https://github.com/kennethreitz/requests>`_.

For example, if you know that the file you're opening has JSON content:

.. code-block:: pycon

    >>> api.files['com~apple~Notes']['Documents']['information.json'].open().json()
    {'How much we love you': 'lots'}
    >>> api.files['com~apple~Notes']['Documents']['information.json'].open().json()['How much we love you']
    'lots'

Or, if you're downloading a particularly large file, you may want to use the ``stream`` keyword argument, and read directly from the raw response object:

.. code-block:: pycon

    >>> download = api.files['com~apple~Notes']['Documents']['big_file.zip'].open(stream=True)
    >>> with open('downloaded_file.zip', 'wb') as opened_file:
            opened_file.write(download.raw.read())

File Storage (iCloud Drive)
===========================

You can access your iCloud Drive using an API identical to the Ubiquity one described in the previous section, except that it is rooted at ```api.drive```:

.. code-block:: pycon

    >>> api.drive.dir()
    ['Holiday Photos', 'Work Files']
    >>> api.drive['Holiday Photos']['2013']['Sicily'].dir()
    ['DSC08116.JPG', 'DSC08117.JPG']

    >>> drive_file = api.drive['Holiday Photos']['2013']['Sicily']['DSC08116.JPG']
    >>> drive_file.name
    'DSC08116.JPG'
    >>> drive_file.date_modified
    datetime.datetime(2013, 3, 21, 12, 28, 12) # NB this is UTC
    >>> drive_file.size
    2021698
    >>> drive_file.type
    'file'

The ``open`` method will return a response object from which you can read the file's contents:

.. code-block:: python

        from shutil import copyfileobj
        with drive_file.open(stream=True) as response:
            with open(drive_file.name, 'wb') as file_out:
                copyfileobj(response.raw, file_out)

To interact with files and directions the ``mkdir``, ``rename`` and ``delete`` functions are available
for a file or folder:

.. code-block:: python

    api.drive['Holiday Photos'].mkdir('2020')
    api.drive['Holiday Photos']['2020'].rename('2020_copy')
    api.drive['Holiday Photos']['2020_copy'].delete()

The ``upload`` method can be used to send a file-like object to the iCloud Drive:

.. code-block:: python

    with open('Vacation.jpeg', 'rb') as file_in:
        api.drive['Holiday Photos'].upload(file_in)

It is strongly suggested to open file handles as binary rather than text to prevent decoding errors
further down the line.

Photo Library
=======================

You can access the iCloud Photo Library through the ``photos`` property.

.. code-block:: pycon

    >>> api.photos.all
    <PhotoAlbum: 'All Photos'>

Individual albums are available through the ``albums`` property:

.. code-block:: pycon

    >>> api.photos.albums['Screenshots']
    <PhotoAlbum: 'Screenshots'>

Which you can iterate to access the photo assets.  The 'All Photos' album is sorted by `added_date` so the most recently added photos are returned first.  All other albums are sorted by `asset_date` (which represents the exif date) :

.. code-block:: pycon

    >>> for photo in api.photos.albums['Screenshots']:
            print(photo, photo.filename)
    <PhotoAsset: id=AVbLPCGkp798nTb9KZozCXtO7jds> IMG_6045.JPG

To download a photo use the `download` method, which will return a `response object <http://www.python-requests.org/en/latest/api/#classes>`_, initialized with ``stream`` set to ``True``, so you can read from the raw response object:

.. code-block:: python

    photo = next(iter(api.photos.albums['Screenshots']), None)
    download = photo.download()
    with open(photo.filename, 'wb') as opened_file:
        opened_file.write(download.raw.read())

Note: Consider using ``shutil.copyfile`` or another buffered strategy for downloading the file so that the whole file isn't read into memory before writing.

Information about each version can be accessed through the ``versions`` property:

.. code-block:: pycon

    >>> photo.versions.keys()
    ['medium', 'original', 'thumb']

To download a specific version of the photo asset, pass the version to ``download()``:

.. code-block:: python

    download = photo.download('thumb')
    with open(photo.versions['thumb']['filename'], 'wb') as thumb_file:
        thumb_file.write(download.raw.read())

To delete a photo asset, call the delete method
>>> photo.delete()
Or use the delete many option (uses one REST call)
>>> api.photos.delete_many([photo1.asset_id, photo2.asset_id])

To upload an image
>>> api.photos.upload_file(file_path)


Code samples
============

If you wanna see some code samples see the `code samples file </CODE_SAMPLES.md>`_.
