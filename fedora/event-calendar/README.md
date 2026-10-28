# Dato-like panel clock: Event Calendar

On macOS, [Dato](https://sindresorhus.com/dato) puts your next meeting, a Join button and other time zones in the menu bar. On KDE Plasma 6 the free [Event Calendar](https://github.com/divinae/plasma-applet-eventcalendar) widget does the same: it replaces the panel clock, shows "in 5 min" for the next meeting, adds Join buttons for Meet, Zoom and Teams, and has agenda, day, week and month views plus extra time zones.

If you only want other time zones, Plasma's own Digital Clock already does that: right-click it → Configure → Time Zones.

## Install

Skip the repo's `install` script. It runs `sudo dnf install -y` on a list that includes Akonadi and KDE PIM packages, without showing you what it will install. All the widget needs on Fedora 44 KDE is already there except one Python package.

```bash
git clone https://github.com/divinae/plasma-applet-eventcalendar.git
cd plasma-applet-eventcalendar
kpackagetool6 -t Plasma/Applet -i package

# recurring events in .ics calendars; Fedora doesn't package this, and pip isn't installed by default
python3 -m venv /tmp/pipvenv
/tmp/pipvenv/bin/pip install --target ~/.local/share/plasma_org.kde.plasma.eventcalendar/python 'recurring-ical-events>=3.8,<4'
rm -rf /tmp/pipvenv
```

Then right-click the panel → Add Widgets → "Event Calendar", and drag it next to (or in place of) the clock.

## Which calendar to connect

- **iCal address (read-only):** in Google Calendar's settings for your calendar, copy "Secret address in iCal format" and add it on the widget's **ICalendar (.ics)** page. The widget can only read it. Keep the address private; anyone with it can see your calendar.
- **Google sign-in:** the widget's Google Calendar page gives it permission to create, edit and delete events. It only writes when you change an event in it, but KDE's own Google sync (Akonadi) once deleted events on Google, so read-only is the safer choice. If you sign in and later stop using it, also remove its access at myaccount.google.com/connections.

## Known problem: resizing the popup can freeze Plasma

Dragging the popup's edge to resize it froze plasmashell (two cores busy, panel dead). To recover:

```bash
pkill -9 plasmashell; systemctl --user start plasma-plasmashell
```

If it freezes again on start, set a smaller saved size before starting it. The widget's section in `~/.config/plasma-org.kde.plasma.desktop-appletsrc` has `popupWidth` and `popupHeight`; 560 and 600 work.

## Update and remove

```bash
cd plasma-applet-eventcalendar && git pull && kpackagetool6 -t Plasma/Applet -u package
```

```bash
kpackagetool6 -t Plasma/Applet -r org.kde.plasma.eventcalendar
rm -rf ~/.local/share/plasma_org.kde.plasma.eventcalendar
```

Tested with Event Calendar 1.8.1 on Plasma 6.7.5, Fedora 44.
