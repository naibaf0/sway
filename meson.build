project(
	'sway',
	'c',
	license: 'MIT',
	default_options: [
		'c_std=c11',
		'warning_level=2',
		'werror=true',
	],
)

add_project_arguments('-Wno-unused-parameter', language: 'c')
add_project_arguments('-Wno-unused-function', language: 'c')
add_project_arguments('-Wno-unused-result', language: 'c')
add_project_arguments('-DWL_HIDE_DEPRECATED', language: 'c')
add_project_arguments('-DWLR_USE_UNSTABLE', language: 'c')

cc = meson.get_compiler('c')

is_freebsd = host_machine.system().startswith('freebsd')
datadir = get_option('datadir')
sysconfdir = get_option('sysconfdir')
prefix = get_option('prefix')

if is_freebsd
	add_project_arguments('-D_C11_SOURCE', language: 'c')
endif

swayidle_deps = []

jsonc          = dependency('json-c', version: '>=0.13')
pcre           = dependency('libpcre')
wlroots        = dependency('wlroots', fallback: ['wlroots', 'wlroots'])
wayland_server = dependency('wayland-server')
wayland_client = dependency('wayland-client')
wayland_cursor = dependency('wayland-cursor')
wayland_egl    = dependency('wayland-egl')
wayland_protos = dependency('wayland-protocols', version: '>=1.14')
xkbcommon      = dependency('xkbcommon')
cairo          = dependency('cairo')
pango          = dependency('pango')
pangocairo     = dependency('pangocairo')
gdk_pixbuf     = dependency('gdk-pixbuf-2.0', required: false)
pixman         = dependency('pixman-1')
libinput       = dependency('libinput', version: '>=1.6.0')
libpam         = cc.find_library('pam', required: false)
crypt          = cc.find_library('crypt', required: false)
systemd        = dependency('libsystemd', required: false)
elogind        = dependency('libelogind', required: false)
math           = cc.find_library('m')
rt             = cc.find_library('rt')
git            = find_program('git', required: false)

conf_data = configuration_data()

if get_option('enable-xwayland')
	conf_data.set('HAVE_XWAYLAND', true)
	xcb = dependency('xcb')
else
	conf_data.set('HAVE_XWAYLAND', false)
endif

if gdk_pixbuf.found()
	conf_data.set('HAVE_GDK_PIXBUF', true)
endif

if systemd.found()
	conf_data.set('SWAY_IDLE_HAS_SYSTEMD', true)
	swayidle_deps += systemd
endif

if elogind.found()
	conf_data.set('SWAY_IDLE_HAS_ELOGIND', true)
	swayidle_deps += elogind
endif

if not systemd.found() and not elogind.found()
    warning('The sway binary must be setuid when compiled without (e)logind')
    warning('You must do this manually post-install: chmod a+s /path/to/sway')
endif

scdoc = find_program('scdoc', required: false)

if scdoc.found()
	sh = find_program('sh')
	mandir = get_option('mandir')
	man_files = [
		'sway/sway.1.scd',
		'sway/sway.5.scd',
		'sway/sway-bar.5.scd',
		'sway/sway-input.5.scd',
		'sway/sway-output.5.scd',
		'swaylock/swaylock.1.scd',
		'swaymsg/swaymsg.1.scd',
		'swayidle/swayidle.1.scd',
		'swaynag/swaynag.1.scd',
		'swaynag/swaynag.5.scd',
	]
	foreach filename : man_files
		topic = filename.split('.')[-3].split('/')[-1]
		section = filename.split('.')[-2]
		output = '@0@.@1@'.format(topic, section)

		custom_target(
			output,
			input: filename,
			output: output,
			command: [
				sh, '-c', '@0@ < @INPUT@ > @1@'.format(scdoc.path(), output)
			],
			install: true,
			install_dir: '@0@/man@1@'.format(mandir, section)
		)
	endforeach
endif

# If prefix is '/usr', sysconfdir will be explicitly set to '/etc' by Meson to
# enforce FHS compliance, so we should look for configs there as well.
if prefix == '/usr'
	add_project_arguments('-DSYSCONFDIR="/@0@"'.format(sysconfdir), language : 'c')
else
	add_project_arguments('-DSYSCONFDIR="/@0@/@1@"'.format(prefix, sysconfdir), language : 'c')
endif

version = get_option('sway-version')
if version != ''
	version = '"@0@"'.format(version)
else
	if not git.found()
		error('git is required to make the version string')
	endif

	git_commit_hash = run_command([git.path(), 'describe', '--always', '--tags']).stdout().strip()
	git_branch = run_command([git.path(), 'rev-parse', '--abbrev-ref', 'HEAD']).stdout().strip()
	version = '"@0@ (" __DATE__ ", branch \'@1@\')"'.format(git_commit_hash, git_branch)
endif
add_project_arguments('-DSWAY_VERSION=@0@'.format(version), language: 'c')

if get_option('use_rpath')
	if get_option('custom_rpath') == ''
		# default to platform specific libdir, one level up from the binary
		rpathdir = join_paths('$ORIGIN', '..', '$LIB')
	else
		rpathdir = get_option('custom_rpath')
	endif
else
	rpathdir = ''
endif

sway_inc = include_directories('include')

subdir('include')
subdir('protocols')
subdir('common')
subdir('sway')
subdir('swaymsg')

subdir('client')
subdir('swaybg')
subdir('swaybar')
subdir('swayidle')
subdir('swaynag')
subdir('swaylock')

config = configuration_data()
config.set('datadir', join_paths(prefix, datadir))
config.set('prefix', prefix)

# If prefix is '/usr', sysconfdir will be explicitly set to '/etc' by Meson to
# enforce FHS compliance, so we should look for configs there as well.
if prefix == '/usr'
	config.set('sysconfdir', sysconfdir)
else
	config.set('sysconfdir', join_paths(prefix, sysconfdir))
endif

configure_file(
	configuration: config,
	input: 'config.in',
	output: '@BASENAME@',
	install_dir: sysconfdir + '/sway'
)

if is_freebsd
	configure_file(
		configuration: config,
		input: 'security.d/10-freebsd.in',
		output: '@BASENAME@',
		install_dir: sysconfdir + '/sway/security.d'
	)
else
	configure_file(
		configuration: config,
		input: 'security.d/00-defaults.in',
		output: '@BASENAME@',
		install_dir: sysconfdir + '/sway/security.d'
	)
endif

install_data(
	'sway.desktop',
	install_dir: datadir + '/wayland-sessions'
)

if (get_option('default-wallpaper'))
	wallpaper_files = files(
		'assets/Sway_Wallpaper_Blue_768x1024.png',
		'assets/Sway_Wallpaper_Blue_768x1024_Portrait.png',
		'assets/Sway_Wallpaper_Blue_1136x640.png',
		'assets/Sway_Wallpaper_Blue_1136x640_Portrait.png',
		'assets/Sway_Wallpaper_Blue_1366x768.png',
		'assets/Sway_Wallpaper_Blue_1920x1080.png',
		'assets/Sway_Wallpaper_Blue_2048x1536.png',
		'assets/Sway_Wallpaper_Blue_2048x1536_Portrait.png',
	)
	wallpaper_install_dir = datadir + '/backgrounds/sway'

	install_data(wallpaper_files, install_dir: wallpaper_install_dir)
endif

if (get_option('zsh-completions'))
	zsh_files = files(
		'completions/zsh/_sway',
		'completions/zsh/_swaylock',
		'completions/zsh/_swaymsg',
	)
	zsh_install_dir = datadir + '/zsh/site-functions'

	install_data(zsh_files, install_dir: zsh_install_dir)
endif

if (get_option('bash-completions'))
	bash_files = files(
		'completions/bash/sway',
		'completions/bash/swayidle',
		'completions/bash/swaylock',
		'completions/bash/swaymsg',
	)
	bash_install_dir = datadir + '/bash-completion/completions'

	install_data(bash_files, install_dir: bash_install_dir)
endif

if (get_option('fish-completions'))
	fish_files = files(
		'completions/fish/sway.fish',
		'completions/fish/swayidle.fish',
		'completions/fish/swaylock.fish',
		'completions/fish/swaymsg.fish',
		'completions/fish/swaynag.fish',
	)
	fish_install_dir = datadir + '/fish/completions'

	install_data(fish_files, install_dir: fish_install_dir)
endif
