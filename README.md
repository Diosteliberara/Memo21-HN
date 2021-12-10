# Memo21-HN
git remote add origin https://github.com/Diosteliberara/freewolrd.git git branch -M main git push -u origin main
help me i want the free world 
quiero decir que todos seamos justos y iguales sin desigualdad 



-1:Noinstalado:instalado:)
￼

			
	File	File	
11/11/2009 Shawn	1	1	// Copyright (C) 2008 The Android Open Source Project
11/11/2009 Shawn	2	2	//
11/11/2009 Shawn	3	3	// Licensed under the Apache License, Version 2.0 (the "License");
11/11/2009 Shawn	4	4	// you may not use this file except in compliance with the License.
11/11/2009 Shawn	5	5	// You may obtain a copy of the License at
11/11/2009 Shawn	6	6	//
11/11/2009 Shawn	7	7	// http://www.apache.org/licenses/LICENSE-2.0
11/11/2009 Shawn	8	8	//
11/11/2009 Shawn	9	9	// Unless required by applicable law or agreed to in writing, software
11/11/2009 Shawn	10	10	// distributed under the License is distributed on an "AS IS" BASIS,
11/11/2009 Shawn	11	11	// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
11/11/2009 Shawn	12	12	// See the License for the specific language governing permissions and
11/11/2009 Shawn	13	13	// limitations under the License.
11/11/2009 Shawn	14	14	
11/11/2009 Shawn	15	15	package com.google.gerrit.server.project;
11/11/2009 Shawn	16	16	
6/3/2020 Patrick	17	17	import static com.google.gerrit.server.project.ProjectCache.illegalState;
19/10/2016 Dave	18	18	import static java.util.stream.Collectors.toSet;
19/10/2016 Dave	19	19	
22/1/2019 Paladox	20	20	import com.google.common.annotations.VisibleForTesting;
24/5/2012 Shawn	21	21	import com.google.common.cache.CacheLoader;
24/5/2012 Shawn	22	22	import com.google.common.cache.LoadingCache;
10/1/2018 Dave	23	23	import com.google.common.collect.ImmutableSet;
10/1/2018 Dave	24	24	import com.google.common.collect.ImmutableSortedSet;
24/5/2012 Shawn	25	25	import com.google.common.collect.Sets;
5/6/2018 Edwin	26	26	import com.google.common.flogger.FluentLogger;
9/3/2020 Patrick	27	27	import com.google.gerrit.common.Nullable;
15/10/2019 David	28	28	import com.google.gerrit.entities.AccountGroup;
15/10/2019 David	29	29	import com.google.gerrit.entities.Project;
4/3/2020 Patrick	30	30	import com.google.gerrit.exceptions.StorageException;
19/3/2018 Dave	31	31	import com.google.gerrit.index.project.ProjectIndexer;
2/11/2017 Edwin	32	32	import com.google.gerrit.lifecycle.LifecycleModule;
25/10/2018 Edwin	33	33	import com.google.gerrit.metrics.Description;
25/10/2018 Edwin	34	34	import com.google.gerrit.metrics.Description.Units;
25/10/2018 Edwin	35	35	import com.google.gerrit.metrics.MetricMaker;
25/10/2018 Edwin	36	36	import com.google.gerrit.metrics.Timer0;
11/11/2009 Shawn	37	37	import com.google.gerrit.server.cache.CacheModule;
16/6/2011 Shawn	38	38	import com.google.gerrit.server.config.AllProjectsName;
25/3/2014 Edwin	39	39	import com.google.gerrit.server.config.AllUsersName;
19/5/2011 Shawn	40	40	import com.google.gerrit.server.git.GitRepositoryManager;
12/7/2019 Edwin	41	41	import com.google.gerrit.server.logging.Metadata;
28/9/2018 Edwin	42	42	import com.google.gerrit.server.logging.TraceContext;
28/9/2018 Edwin	43	43	import com.google.gerrit.server.logging.TraceContext.TraceTimer;
30/9/2014 Saša	44	44	import com.google.inject.Inject;
11/11/2009 Shawn	45	45	import com.google.inject.Module;
2/10/2017 Xin	46	46	import com.google.inject.Provider;
11/11/2009 Shawn	47	47	import com.google.inject.Singleton;
11/11/2009 Shawn	48	48	import com.google.inject.TypeLiteral;
11/11/2009 Shawn	49	49	import com.google.inject.name.Named;
10/5/2013 Colby	50	50	import java.io.IOException;
19/10/2016 Dave	51	51	import java.util.Objects;
6/3/2020 Patrick	52	52	import java.util.Optional;
18/1/2013 Shawn	53	53	import java.util.Set;
24/5/2012 Shawn	54	54	import java.util.concurrent.ExecutionException;
19/5/2011 Shawn	55	55	import java.util.concurrent.locks.Lock;
19/5/2011 Shawn	56	56	import java.util.concurrent.locks.ReentrantLock;
6/2/2017 Dave	57	57	import org.eclipse.jgit.errors.RepositoryNotFoundException;
6/2/2017 Dave	58	58	import org.eclipse.jgit.lib.Repository;
11/11/2009 Shawn	59	59	
11/11/2009 Shawn	60	60	/** Cache of project information, including access rights. */
11/11/2009 Shawn	61	61	@Singleton
11/11/2009 Shawn	62	62	public class ProjectCacheImpl implements ProjectCache {
5/6/2018 Edwin	63	63	private static final FluentLogger logger = FluentLogger.forEnclosingClass();
24/5/2012 Shawn	64	64	
25/4/2018 Dave	65	65	public static final String CACHE_NAME = "projects";
25/4/2018 Dave	66	66	
19/5/2011 Shawn	67	67	private static final String CACHE_LIST = "project_list";
11/11/2009 Shawn	68	68	
11/11/2009 Shawn	69	69	public static Module module() {
11/11/2009 Shawn	70	70	return new CacheModule() {
11/11/2009 Shawn	71	71	@Override
11/11/2009 Shawn	72	72	protected void configure() {
6/2/2017 Dave	73	73	cache(CACHE_NAME, String.class, ProjectState.class).loader(Loader.class);
19/5/2011 Shawn	74	74	
10/1/2018 Dave	75	75	cache(CACHE_LIST, ListKey.class, new TypeLiteral<ImmutableSortedSet<Project.NameKey>>() {})
6/2/2017 Dave	76	76	.maximumWeight(1)
6/2/2017 Dave	77	77	.loader(Lister.class);
19/5/2011 Shawn	78	78	
11/11/2009 Shawn	79	79	bind(ProjectCacheImpl.class);
11/11/2009 Shawn	80	80	bind(ProjectCache.class).to(ProjectCacheImpl.class);
2/11/2017 Edwin	81	81	
2/11/2017 Edwin	82	82	install(
2/11/2017 Edwin	83	83	new LifecycleModule() {
2/11/2017 Edwin	84	84	@Override
2/11/2017 Edwin	85	85	protected void configure() {
2/11/2017 Edwin	86	86	listener().to(ProjectCacheWarmer.class);
2/11/2017 Edwin	87	87	listener().to(ProjectCacheClock.class);
2/11/2017 Edwin	88	88	}
2/11/2017 Edwin	89	89	});
11/11/2009 Shawn	90	90	}
11/11/2009 Shawn	91	91	};
11/11/2009 Shawn	92	92	}
11/11/2009 Shawn	93	93	
16/6/2011 Shawn	94	94	private final AllProjectsName allProjectsName;
25/3/2014 Edwin	95	95	private final AllUsersName allUsersName;
24/5/2012 Shawn	96	96	private final LoadingCache<String, ProjectState> byName;
10/1/2018 Dave	97	97	private final LoadingCache<ListKey, ImmutableSortedSet<Project.NameKey>> list;
19/5/2011 Shawn	98	98	private final Lock listLock;
23/1/2012 Shawn	99	99	private final ProjectCacheClock clock;
2/10/2017 Xin	100	100	private final Provider<ProjectIndexer> indexer;
25/10/2018 Edwin	101	101	private final Timer0 guessRelevantGroupsLatency;
11/11/2009 Shawn	102	102	
11/11/2009 Shawn	103	103	@Inject
16/7/2010 Shawn	104	104	ProjectCacheImpl(
16/6/2011 Shawn	105	105	final AllProjectsName allProjectsName,
25/3/2014 Edwin	106	106	final AllUsersName allUsersName,
24/5/2012 Shawn	107	107	@Named(CACHE_NAME) LoadingCache<String, ProjectState> byName,
10/1/2018 Dave	108	108	@Named(CACHE_LIST) LoadingCache<ListKey, ImmutableSortedSet<Project.NameKey>> list,
2/10/2017 Xin	109	109	ProjectCacheClock clock,
25/10/2018 Edwin	110	110	Provider<ProjectIndexer> indexer,
25/10/2018 Edwin	111	111	MetricMaker metricMaker) {
16/6/2011 Shawn	112	112	this.allProjectsName = allProjectsName;
25/3/2014 Edwin	113	113	this.allUsersName = allUsersName;
16/7/2010 Shawn	114	114	this.byName = byName;
19/5/2011 Shawn	115	115	this.list = list;
19/5/2011 Shawn	116	116	this.listLock = new ReentrantLock(true /* fair */);
23/1/2012 Shawn	117	117	this.clock = clock;
2/10/2017 Xin	118	118	this.indexer = indexer;
25/10/2018 Edwin	119	119	
25/10/2018 Edwin	120	120	this.guessRelevantGroupsLatency =
25/10/2018 Edwin	121	121	metricMaker.newTimer(
25/10/2018 Edwin	122	122	"group/guess_relevant_groups_latency",
25/10/2018 Edwin	123	123	new Description("Latency for guessing relevant groups")
25/10/2018 Edwin	124	124	.setCumulative()
25/10/2018 Edwin	125	125	.setUnit(Units.NANOSECONDS));
11/11/2009 Shawn	126	126	}
11/11/2009 Shawn	127	127	
16/6/2011 Shawn	128	128	@Override
16/6/2011 Shawn	129	129	public ProjectState getAllProjects() {
6/3/2020 Patrick	130	130	return get(allProjectsName).orElseThrow(illegalState(allProjectsName));
16/6/2011 Shawn	131	131	}
16/6/2011 Shawn	132	132	
10/5/2013 Colby	133	133	@Override
25/3/2014 Edwin	134	134	public ProjectState getAllUsers() {
6/3/2020 Patrick	135	135	return get(allUsersName).orElseThrow(illegalState(allUsersName));
25/3/2014 Edwin	136	136	}
25/3/2014 Edwin	137	137	
25/3/2014 Edwin	138	138	@Override
9/3/2020 Patrick	139	139	public Optional<ProjectState> get(@Nullable Project.NameKey projectName) {
9/3/2020 Patrick	140	140	if (projectName == null) {
9/3/2020 Patrick	141	141	return Optional.empty();
9/3/2020 Patrick	142	142	}
9/3/2020 Patrick	143	143	
6/2/2017 Dave	144	144	try {
9/3/2020 Patrick	145	145	ProjectState state = byName.get(projectName.get());
9/3/2020 Patrick	146	146	if (state != null && state.needsRefresh(clock.read())) {
9/3/2020 Patrick	147	147	byName.invalidate(projectName.get());
9/3/2020 Patrick	148	148	state = byName.get(projectName.get());
9/3/2020 Patrick	149	149	}
9/3/2020 Patrick	150	150	return Optional.of(state);
9/3/2020 Patrick	151	151	} catch (Exception e) {
9/3/2020 Patrick	152	152	if ((e.getCause() instanceof RepositoryNotFoundException)) {
9/3/2020 Patrick	153	153	logger.atFine().log("Cannot find project %s", projectName.get());
9/3/2020 Patrick	154	154	return Optional.empty();
9/3/2020 Patrick	155	155	}
13/3/2020 Edwin	156	156	throw new StorageException(
13/3/2020 Edwin	157	157	String.format("project state of project %s not available", projectName.get()), e);
10/5/2013 Colby	158	158	}
10/5/2013 Colby	159	159	}
10/5/2013 Colby	160	160	
10/5/2013 Colby	161	161	@Override
		162	public void evict(Project.NameKey p) {
		163	if (p != null) {
		164	logger.atFine().log("Evict project '%s'", p.get());
		165	byName.invalidate(p.get());
		166	}
		167	}
		168	
		169	@Override
17/11/2021 Adithya	162	170	public void evictAndReindex(Project p) {
17/11/2021 Adithya	163	171	evictAndReindex(p.getNameKey());
11/11/2009 Shawn	164	172	}
23/4/2010 lincoln	165	173	
29/10/2014 Dave	166	174	@Override
17/11/2021 Adithya	167	175	public void evictAndReindex(Project.NameKey p) {
10/5/2013 Edwin	168		if (p != null) {
27/8/2018 Edwin	169		logger.atFine().log("Evict project '%s'", p.get());
10/5/2013 Edwin	170		byName.invalidate(p.get());
10/5/2013 Edwin	171		}
		176	evict(p);
2/10/2017 Xin	172	177	indexer.get().index(p);
10/5/2013 Edwin	173	178	}
10/5/2013 Edwin	174	179	
19/5/2011 Shawn	175	180	@Override
11/3/2020 Edwin	176	181	public void remove(Project p) {
14/1/2018 David	177	182	remove(p.getNameKey());
14/1/2018 David	178	183	}
14/1/2018 David	179	184	
14/1/2018 David	180	185	@Override
11/3/2020 Edwin	181	186	public void remove(Project.NameKey name) {
			
	+136 common line		
		

blob: 573dd44683524c423f2cc7637a83623ab51eb139 [file] [log] [blame]
#!/usr/bin/env python3
# Copyright 2019 The Android Open Source Project
#2º1º :ArribaHonduras
# Licensed under the Apache License, Version 2.0 (the "License"); MemoYEmilylove
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#      http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

"""Wrapper to run pytest with the right settings."""

import errno
import os
import shutil
import subprocess
import sys


def find_pytest():
"""Try to locate a good version of pytest."""
# If we're in a virtualenv, assume that it's provided the right pytest.
if 'VIRTUAL_ENV' in os.environ:
return 'pytest'

# Use the Python 3 version if available.
ret = shutil.which('pytest-3')
if ret:
return ret

# Hopefully this is a Python 3 version.
ret = shutil.which('pytest')
if ret:
return ret

print('%s: unable to find pytest.' % (__file__,), file=sys.stderr)
print('%s: Try installing: sudo apt-get install python-pytest' % (__file__,),
file=sys.stderr)


def main(argv):
"""The main entry."""
# Add the repo tree to PYTHONPATH as the tests expect to be able to import
# modules directly.
pythonpath = os.path.dirname(os.path.realpath(__file__))
oldpythonpath = os.environ.get('PYTHONPATH', None)
if oldpythonpath is not None:
pythonpath += os.pathsep + oldpythonpath
os.environ['PYTHONPATH'] = pythonpath

pytest = find_pytest()
return subprocess.run([pytest] + argv, check=False).returncode


if __name__ == '__main__':
sys.exit(main(sys.argv[1:]))

default.xml
blob: 7c2f7abc720e5127a9380eab3cebca65efc25208 [file] [log] [blame]
<?xml version="1.0" encoding="UTF-8"?>
<manifest>

<remote  name="aosp"
fetch=".." />
<default revision="master"
remote="aosp"
sync-j="4" />

<project name="accessories/manifest" />
<project name="assets/android-studio-ux-assets" />
<project name="brillo/manifest" />
<project name="device/amlogic/yukawa" />
<project name="device/amlogic/yukawa-kernel" />
<project name="device/asus/deb" />
<project name="device/asus/flo" />
<project name="device/asus/flo-kernel" />
<project name="device/asus/fugu" />
<project name="device/asus/fugu-kernel" />
<project name="device/asus/grouper" />
<project name="device/asus/tilapia" />
<project name="device/casio/koi-uboot" />
<project name="device/common" />
<project name="device/freescale/picoimx" />
<project name="device/generic/arm64" />
<project name="device/generic/armv7-a" />
<project name="device/generic/armv7-a-neon" />
<project name="device/generic/art" />
<project name="device/generic/brillo" />
<project name="device/generic/car" />
<project name="device/generic/common" />
<project name="device/generic/goldfish" />
<project name="device/generic/goldfish-opengl" />
<project name="device/generic/mini-emulator-arm64" />
<project name="device/generic/mini-emulator-armv7-a-neon" />
<project name="device/generic/mini-emulator-mips" />
<project name="device/generic/mini-emulator-mips64" />
<project name="device/generic/mini-emulator-x86" />
<project name="device/generic/mini-emulator-x86_64" />
<project name="device/generic/mips" />
<project name="device/generic/mips64" />
<project name="device/generic/opengl-transport" />
<project name="device/generic/qemu" />
<project name="device/generic/trusty" />
<project name="device/generic/uml" />
<project name="device/generic/vulkan-cereal" />
<project name="device/generic/x86" />
<project name="device/generic/x86_64" />
<project name="device/google/accessory/adk2012" />
<project name="device/google/accessory/adk2012_demo" />
<project name="device/google/accessory/arduino" />
<project name="device/google/accessory/demokit" />
<project name="device/google/atv" />
<project name="device/google/barbet" />
<project name="device/google/barbet-kernel" />
<project name="device/google/barbet-sepolicy" />
<project name="device/google/bonito" />
<project name="device/google/bonito-kernel" />
<project name="device/google/bonito-sepolicy" />
<project name="device/google/bramble" />
<project name="device/google/bramble-kernel" />
<project name="device/google/bramble-sepolicy" />
<project name="device/google/contexthub" />
<project name="device/google/coral" />
<project name="device/google/coral-kernel" />
<project name="device/google/coral-sepolicy" />
<project name="device/google/crosshatch" />
<project name="device/google/crosshatch-kernel" />
<project name="device/google/crosshatch-sepolicy" />
<project name="device/google/cuttlefish" />
<project name="device/google/cuttlefish_common" />
<project name="device/google/cuttlefish_kernel" />
<project name="device/google/cuttlefish_prebuilts" />
<project name="device/google/cuttlefish_vmm" />
<project name="device/google/debugcable" />
<project name="device/google/dragon" />
<project name="device/google/dragon-kernel" />
<project name="device/google/fuchsia" />
<project name="device/google/gs-common" />
<project name="device/google/gs101" />
<project name="device/google/gs101-sepolicy" />
<project name="device/google/marlin" />
<project name="device/google/marlin-kernel" />
<project name="device/google/muskie" />
<project name="device/google/phantasm" />
<project name="device/google/raviole" />
<project name="device/google/raviole-kernel" />
<project name="device/google/redbull" />
<project name="device/google/redbull-kernel" />
<project name="device/google/redbull-sepolicy" />
<project name="device/google/redfin" />
<project name="device/google/redfin-kernel" />
<project name="device/google/redfin-sepolicy" />
<project name="device/google/sunfish" />
<project name="device/google/sunfish-kernel" />
<project name="device/google/sunfish-sepolicy" />
<project name="device/google/taimen" />
<project name="device/google/trout" />
<project name="device/google/vrservices" />
<project name="device/google/wahoo" />
<project name="device/google/wahoo-kernel" />
<project name="device/google_car" />
<project name="device/htc/common" />
<project name="device/htc/dream" />
<project name="device/htc/dream-sapphire" />
<project name="device/htc/flounder" />
<project name="device/htc/flounder-kernel" />
<project name="device/htc/passion" />
<project name="device/htc/passion-common" />
<project name="device/htc/sapphire" />
<project name="device/huawei/angler" />
<project name="device/huawei/angler-kernel" />
<project name="device/imagination/creatorci41" />
<project name="device/intel/edison" />
<project name="device/intel/minnowboard" />
<project name="device/lge/bullhead" />
<project name="device/lge/bullhead-kernel" />
<project name="device/lge/hammerhead" />
<project name="device/lge/hammerhead-kernel" />
<project name="device/lge/mako" />
<project name="device/lge/mako-kernel" />
<project name="device/linaro/bootloader/OpenPlatformPkg" />
<project name="device/linaro/bootloader/arm-trusted-firmware" />
<project name="device/linaro/bootloader/edk2" />
<project name="device/linaro/dragonboard" />
<project name="device/linaro/dragonboard-kernel" />
<project name="device/linaro/hikey" />
<project name="device/linaro/hikey-kernel" />
<project name="device/linaro/poplar" />
<project name="device/linaro/poplar-kernel" />
<project name="device/mediatek/wembley-sepolicy" />
<project name="device/moto/common" />
<project name="device/moto/shamu" />
<project name="device/moto/shamu-kernel" />
<project name="device/moto/stingray" />
<project name="device/moto/wingray" />
<project name="device/pifoundation/rpi3" />
<project name="device/qcom/dragonboard" />
<project name="device/rockchip/kylin" />
<project name="device/sample" />
<project name="device/samsung/crespo" />
<project name="device/samsung/crespo4g" />
<project name="device/samsung/maguro" />
<project name="device/samsung/manta" />
<project name="device/samsung/toro" />
<project name="device/samsung/toroplus" />
<project name="device/samsung/torospr" />
<project name="device/samsung/tuna" />
<project name="device/samsung_slsi/arndale" />
<project name="device/sony/lt26" />
<project name="device/ti/beagle-x15" />
<project name="device/ti/beagle-x15-kernel" />
<project name="device/ti/bootloader/uboot" />
<project name="device/ti/panda" />
<project name="kernel/amlogic-tv-modules/dhd-driver" />
<project name="kernel/amlogic-tv-modules/mali-driver" />
<project name="kernel/amlogic-tv-modules/media_modules" />
<project name="kernel/amlogic-tv-modules/optee_linuxdriver" />
<project name="kernel/amlogic-tv-modules/wifi_rtl8822BS" />
<project name="kernel/arm64" />
<project name="kernel/build" />
<project name="kernel/common" />
<project name="kernel/common-modules/virtual-device" />
<project name="kernel/common-patches" />
<project name="kernel/configs" />
<project name="kernel/cuttlefish-modules" />
<project name="kernel/exynos" />
<project name="kernel/goldfish" />
<project name="kernel/goldfish-modules" />
<project name="kernel/google-modules/amplifiers" />
<project name="kernel/google-modules/aoc" />
<project name="kernel/google-modules/aoc-ipc" />
<project name="kernel/google-modules/bluetooth/broadcom" />
<project name="kernel/google-modules/bms" />
<project name="kernel/google-modules/display" />
<project name="kernel/google-modules/edgetpu" />
<project name="kernel/google-modules/fingerprint/fpc" />
<project name="kernel/google-modules/fingerprint/goodix" />
<project name="kernel/google-modules/gpu" />
<project name="kernel/google-modules/lwis" />
<project name="kernel/google-modules/nfc" />
<project name="kernel/google-modules/power/reset" />
<project name="kernel/google-modules/touch/common" />
<project name="kernel/google-modules/touch/fts_touch" />
<project name="kernel/google-modules/touch/sec_touch" />
<project name="kernel/google-modules/uwb" />
<project name="kernel/google-modules/wlan/bcmdhd/bcm43752" />
<project name="kernel/google-modules/wlan/bcmdhd/bcm4389" />
<project name="kernel/gs" />
<project name="kernel/hikey-linaro" />
<project name="kernel/hikey-modules" />
<project name="kernel/lk" />
<project name="kernel/manifest" />
<project name="kernel/mediatek" />
<project name="kernel/msm" />
<project name="kernel/msm-extra" path="kernel/msm-extra.git" />
<project name="kernel/msm-extra/camera-devicetree" />
<project name="kernel/msm-extra/camera-kernel" />
<project name="kernel/msm-extra/config" />
<project name="kernel/msm-extra/dataipa" />
<project name="kernel/msm-extra/devicetree" />
<project name="kernel/msm-extra/display-devicetree" />
<project name="kernel/msm-extra/display-drivers" />
<project name="kernel/msm-extra/drivers" />
<project name="kernel/msm-extra/video-driver" />
<project name="kernel/msm-modules" path="kernel/msm-modules.git" />
<project name="kernel/msm-modules/data-kernel" />
<project name="kernel/msm-modules/fts_touch" />
<project name="kernel/msm-modules/fts_touch_s5" />
<project name="kernel/msm-modules/qca-wfi-host-cmn" />
<project name="kernel/msm-modules/qcacld" />
<project name="kernel/msm-modules/sec_touch" />
<project name="kernel/msm-modules/wlan" />
<project name="kernel/msm-modules/wlan-fw-api" />
<project name="kernel/omap" />
<project name="kernel/prebuilts/4.19/arm64" />
<project name="kernel/prebuilts/5.10/arm64" />
<project name="kernel/prebuilts/5.10/x86-64" />
<project name="kernel/prebuilts/5.4/arm64" />
<project name="kernel/prebuilts/5.4/x86-64" />
<project name="kernel/prebuilts/build-tools" />
<project name="kernel/prebuilts/common-modules/virtual-device/4.19/arm64" />
<project name="kernel/prebuilts/common-modules/virtual-device/4.19/x86-64" />
<project name="kernel/prebuilts/common-modules/virtual-device/5.10/arm64" />
<project name="kernel/prebuilts/common-modules/virtual-device/5.10/x86-64" />
<project name="kernel/prebuilts/common-modules/virtual-device/5.4/arm64" />
<project name="kernel/prebuilts/common-modules/virtual-device/5.4/x86-64" />
<project name="kernel/prebuilts/common-modules/virtual-device/mainline/arm64" />
<project name="kernel/prebuilts/common-modules/virtual-device/mainline/x86-64" />
<project name="kernel/prebuilts/mainline/arm64" />
<project name="kernel/samsung" />
<project name="kernel/superproject" />
<project name="kernel/tegra" />
<project name="kernel/tests" />
<project name="kernel/x86" />
<project name="kernel/x86_64" />
<project name="mirror/manifest" />
<project name="platform/abi/cpp" />
<project name="platform/art" />
<project name="platform/bbuildbot_config" />
<project name="platform/bionic" />
<project name="platform/bootable/bootloader/legacy" />
<project name="platform/bootable/diskinstaller" />
<project name="platform/bootable/libbootloader" />
<project name="platform/bootable/recovery" />
<project name="platform/build" path="platform/build.git" />
<project name="platform/build/bazel" />
<project name="platform/build/bazel_common_rules" />
<project name="platform/build/blueprint" />
<project name="platform/build/docker" />
<project name="platform/build/kati" />
<project name="platform/build/pesto" />
<project name="platform/build/soong" />
<project name="platform/compatibility/cdd" />
<project name="platform/cts" />
<project name="platform/dalvik" />
<project name="platform/dalvik-snapshot" />
<project name="platform/dalvik2" />
<project name="platform/developers/build" />
<project name="platform/developers/demos" />
<project name="platform/developers/docs" />
<project name="platform/developers/samples/android" />
<project name="platform/development" />
<project name="platform/docs/source.android.com" />
<project name="platform/external/ARMComputeLibrary" />
<project name="platform/external/AntennaPod/AntennaPod" />
<project name="platform/external/AntennaPod/AudioPlayer" />
<project name="platform/external/AntennaPod/afollestad" />
<project name="platform/external/ComputeLibrary" />
<project name="platform/external/FP16" />
<project name="platform/external/FXdiv" />
<project name="platform/external/GhostAWT" />
<project name="platform/external/ImageMagick" />
<project name="platform/external/Mako" />
<project name="platform/external/Microsoft-GSL" />
<project name="platform/external/Microsoft-unittest-cpp" />
<project name="platform/external/OpenCL-CTS" />
<project name="platform/external/OpenCSD" />
<project name="platform/external/Reactive-Extensions/RxCpp" />
<project name="platform/external/TestParameterInjector" />
<project name="platform/external/XNNPACK" />
<project name="platform/external/aac" />
<project name="platform/external/abi-compliance-checker" />
<project name="platform/external/abi-dumper" />
<project name="platform/external/abseil-cpp" />
<project name="platform/external/actionbarsherlock" />
<project name="platform/external/adeb" />
<project name="platform/external/adhd" />
<project name="platform/external/adt-infra" />
<project name="platform/external/aes" />
<project name="platform/external/alsa-lib" />
<project name="platform/external/android-clat" />
<project name="platform/external/android-cmake" />
<project name="platform/external/android-kotlin-demo" />
<project name="platform/external/android-mock" />
<project name="platform/external/android-nn-driver" />
<project name="platform/external/android-studio-gradle-test" />
<project name="platform/external/androidplot" />
<project name="platform/external/angle" />
<project name="platform/external/annotation-tools" />
<project name="platform/external/ant-glob" />
<project name="platform/external/antlr" />
<project name="platform/external/apache-apr" />
<project name="platform/external/apache-apr-util" />
<project name="platform/external/apache-commons-bcel" />
<project name="platform/external/apache-commons-compress" />
<project name="platform/external/apache-commons-math" />
<project name="platform/external/apache-harmony" />
<project name="platform/external/apache-http" />
<project name="platform/external/apache-log4cxx" />
<project name="platform/external/apache-qp" />
<project name="platform/external/apache-xml" />
<project name="platform/external/apple-coreaudiosamples" />
<project name="platform/external/archive-patcher" />
<project name="platform/external/arduino" />
<project name="platform/external/arduino-ide" />
<project name="platform/external/arm-neon-tests" />
<project name="platform/external/arm-optimized-routines" />
<project name="platform/external/arm-trusted-firmware" />
<project name="platform/external/armnn" />
<project name="platform/external/astc-codec" />
<project name="platform/external/astl" />
<project name="platform/external/auto" />
<project name="platform/external/autotest" />
<project name="platform/external/avahi" />
<project name="platform/external/avb" />
<project name="platform/external/bart" />
<project name="platform/external/bazel-skylib" />
<project name="platform/external/bazelbuild-remote-apis" />
<project name="platform/external/bazelbuild-rules_android" />
<project name="platform/external/bc" />
<project name="platform/external/bcc" />
<project name="platform/external/bison" />
<project name="platform/external/blktrace" />
<project name="platform/external/bloaty" />
<project name="platform/external/bluetooth/bluedroid" />
<project name="platform/external/bluetooth/bluez" />
<project name="platform/external/bluetooth/glib" />
<project name="platform/external/bluetooth/hcidump" />
<project name="platform/external/bluez" />
<project name="platform/external/boost" />
<project name="platform/external/boringssl" />
<project name="platform/external/bouncycastle" />
<project name="platform/external/brotli" />
<project name="platform/external/bsdiff" />
<project name="platform/external/bvb" />
<project name="platform/external/bzip2" />
<project name="platform/external/c-ares" />
<project name="platform/external/caliper" />
<project name="platform/external/capstone" />
<project name="platform/external/catch2" />
<project name="platform/external/cblas" />
<project name="platform/external/cbor-java" />
<project name="platform/external/ceres-solver" />
<project name="platform/external/checkpolicy" />
<project name="platform/external/checkstyle" />
<project name="platform/external/cherry" />
<project name="platform/external/chromite" />
<project name="platform/external/chromium" />
<project name="platform/external/chromium-libpac" />
<project name="platform/external/chromium-trace" />
<project name="platform/external/chromium-webview" />
<project name="platform/external/chromium_org" path="platform/external/chromium_org.git" />
<project name="platform/external/chromium_org/sdch/open-vcdiff" />
<project name="platform/external/chromium_org/testing/gtest" />
<project name="platform/external/chromium_org/third_party/WebKit" />
<project name="platform/external/chromium_org/third_party/angle" />
<project name="platform/external/chromium_org/third_party/angle_dx11" />
<project name="platform/external/chromium_org/third_party/boringssl/src" />
<project name="platform/external/chromium_org/third_party/brotli/src" />
<project name="platform/external/chromium_org/third_party/eyesfree/src/android/java/src/com/googlecode/eyesfree/braille" />
<project name="platform/external/chromium_org/third_party/freetype" />
<project name="platform/external/chromium_org/third_party/icu" />
<project name="platform/external/chromium_org/third_party/leveldatabase/src" />
<project name="platform/external/chromium_org/third_party/libaddressinput/src" />
<project name="platform/external/chromium_org/third_party/libjingle/source/talk" />
<project name="platform/external/chromium_org/third_party/libjpeg_turbo" />
<project name="platform/external/chromium_org/third_party/libphonenumber/src/phonenumbers" />
<project name="platform/external/chromium_org/third_party/libphonenumber/src/resources" />
<project name="platform/external/chromium_org/third_party/libsrtp" />
<project name="platform/external/chromium_org/third_party/libvpx" />
<project name="platform/external/chromium_org/third_party/libyuv" />
<project name="platform/external/chromium_org/third_party/mesa/src" />
<project name="platform/external/chromium_org/third_party/openmax_dl" />
<project name="platform/external/chromium_org/third_party/openssl" />
<project name="platform/external/chromium_org/third_party/opus/src" />
<project name="platform/external/chromium_org/third_party/ots" />
<project name="platform/external/chromium_org/third_party/sfntly/cpp/src" />
<project name="platform/external/chromium_org/third_party/skia" path="platform/external/chromium_org/third_party/skia.git" />
<project name="platform/external/chromium_org/third_party/skia/gyp" />
<project name="platform/external/chromium_org/third_party/skia/include" />
<project name="platform/external/chromium_org/third_party/skia/src" />
<project name="platform/external/chromium_org/third_party/smhasher/src" />
<project name="platform/external/chromium_org/third_party/usrsctp/usrsctplib" />
<project name="platform/external/chromium_org/third_party/webrtc" />
<project name="platform/external/chromium_org/third_party/yasm/source/patched-yasm" />
<project name="platform/external/chromium_org/tools/grit" />
<project name="platform/external/chromium_org/tools/gyp" />
<project name="platform/external/chromium_org/v8" />
<project name="platform/external/cibu-fonts" />
<project name="platform/external/clang" />
<project name="platform/external/clang-tools-extra" />
<project name="platform/external/clang_35a" />
<project name="platform/external/cldr" />
<project name="platform/external/clearsilver" />
<project name="platform/external/cmake" />
<project name="platform/external/cmockery" />
<project name="platform/external/cn-cbor" />
<project name="platform/external/codesourcery" />
<project name="platform/external/collada" />
<project name="platform/external/compiler-rt" />
<project name="platform/external/compiler-rt_35a" />
<project name="platform/external/connectedappssdk" />
<project name="platform/external/conscrypt" />
<project name="platform/external/cpu_features" />
<project name="platform/external/cpuinfo" />
<project name="platform/external/crcalc" />
<project name="platform/external/cros/system_api" />
<project name="platform/external/crosvm" />
<project name="platform/external/cryptsetup" />
<project name="platform/external/curl" />
<project name="platform/external/dagger2" />
<project name="platform/external/dbus" />
<project name="platform/external/dbus-binding-generator" />
<project name="platform/external/deqp" />
<project name="platform/external/deqp-deps/SPIRV-Headers" />
<project name="platform/external/deqp-deps/SPIRV-Tools" />
<project name="platform/external/deqp-deps/amber" />
<project name="platform/external/deqp-deps/glslang" />
<project name="platform/external/desugar" />
<project name="platform/external/devlib" />
<project name="platform/external/dexmaker" />
<project name="platform/external/dhcpcd" />
<project name="platform/external/dhcpcd-6.8.2" />
<project name="platform/external/dlmalloc" />
<project name="platform/external/dng_sdk" />
<project name="platform/external/dnsmasq" />
<project name="platform/external/doclava" />
<project name="platform/external/dokka" />
<project name="platform/external/donuts" />
<project name="platform/external/dosfstools" />
<project name="platform/external/downloader" />
<project name="platform/external/drm_gralloc" />
<project name="platform/external/drm_hwcomposer" />
<project name="platform/external/droiddriver" />
<project name="platform/external/dropbear" />
<project name="platform/external/drrickorang" />
<project name="platform/external/dtc" />
<project name="platform/external/dwarves" />
<project name="platform/external/dynamic_depth" />
<project name="platform/external/e2fsprogs" />
<project name="platform/external/easymock" />
<project name="platform/external/eclipse-basebuilder" />
<project name="platform/external/eclipse-windowbuilder" />
<project name="platform/external/effcee" />
<project name="platform/external/eglib" />
<project name="platform/external/eigen" />
<project name="platform/external/elfcopy" />
<project name="platform/external/elfutils" />
<project name="platform/external/embunit" />
<project name="platform/external/emma" />
<project name="platform/external/epid-sdk" />
<project name="platform/external/erofs-utils" />
<project name="platform/external/error_prone" />
<project name="platform/external/escapevelocity" />
<project name="platform/external/esd" />
<project name="platform/external/ethtool" />
<project name="platform/external/exfatprogs" />
<project name="platform/external/exoplayer" />
<project name="platform/external/expat" />
<project name="platform/external/extfuse" />
<project name="platform/external/eyes-free" />
<project name="platform/external/f2fs-tools" />
<project name="platform/external/faad" />
<project name="platform/external/fastrpc" />
<project name="platform/external/fat32lib" />
<project name="platform/external/fdlibm" />
<project name="platform/external/fec" />
<project name="platform/external/fff" />
<project name="platform/external/ffmpeg" />
<project name="platform/external/fft2d" />
<project name="platform/external/fio" />
<project name="platform/external/firebase-messaging" />
<project name="platform/external/flac" />
<project name="platform/external/flashbench" />
<project name="platform/external/flatbuffers" />
<project name="platform/external/flex" />
<project name="platform/external/fmtlib" />
<project name="platform/external/fonttools" />
<project name="platform/external/free-image" />
<project name="platform/external/freetype" />
<project name="platform/external/fsck_msdos" />
<project name="platform/external/fsverity-utils" />
<project name="platform/external/ganymed-ssh2" />
<project name="platform/external/gcc-demangle" />
<project name="platform/external/gdata" />
<project name="platform/external/gemmlowp" />
<project name="platform/external/genext2fs" />
<project name="platform/external/gentoo/integration" />
<project name="platform/external/gentoo/overlays/gentoo" />
<project name="platform/external/gentoo/portage" />
<project name="platform/external/geojson-jackson" />
<project name="platform/external/geonames" />
<project name="platform/external/gflags" />
<project name="platform/external/giflib" />
<project name="platform/external/glide" />
<project name="platform/external/gmock" />
<project name="platform/external/go-cmp" />
<project name="platform/external/go-creachadair-shell" />
<project name="platform/external/go-creachadair-stringset" />
<project name="platform/external/go-etree" />
<project name="platform/external/go-subcommands" />
<project name="platform/external/golang-groupcache" />
<project name="platform/external/golang-protobuf" />
<project name="platform/external/golang-x-net" />
<project name="platform/external/golang-x-oauth2" />
<project name="platform/external/golang-x-sync" />
<project name="platform/external/golang-x-sys" />
<project name="platform/external/golang-x-text" />
<project name="platform/external/golang-x-tools" />
<project name="platform/external/google-api-go-client" />
<project name="platform/external/google-api-services-storage" />
<project name="platform/external/google-benchmark" />
<project name="platform/external/google-breakpad" />
<project name="platform/external/google-cloud-go" />
<project name="platform/external/google-diff-match-patch" />
<project name="platform/external/google-fonts/arbutus-slab" />
<project name="platform/external/google-fonts/arvo" />
<project name="platform/external/google-fonts/barlow" />
<project name="platform/external/google-fonts/big-shoulders-text" />
<project name="platform/external/google-fonts/carrois-gothic-sc" />
<project name="platform/external/google-fonts/coming-soon" />
<project name="platform/external/google-fonts/cutive-mono" />
<project name="platform/external/google-fonts/dancing-script" />
<project name="platform/external/google-fonts/fraunces" />
<project name="platform/external/google-fonts/karla" />
<project name="platform/external/google-fonts/lato" />
<project name="platform/external/google-fonts/lustria" />
<project name="platform/external/google-fonts/rubik" />
<project name="platform/external/google-fonts/source-sans-pro" />
<project name="platform/external/google-fonts/zilla-slab" />
<project name="platform/external/google-fruit" />
<project name="platform/external/google-java-format" />
<project name="platform/external/google-styleguide" />
<project name="platform/external/google-tv-pairing-protocol" />
<project name="platform/external/google-uuid" />
<project name="platform/external/googleapis" />
<project name="platform/external/googleapis-gax-go" />
<project name="platform/external/googleapis-go-genproto" />
<project name="platform/external/googleclient" />
<project name="platform/external/googletest" />
<project name="platform/external/gptfdisk" />
<project name="platform/external/gradle-perf-android-large" />
<project name="platform/external/gradle-perf-android-medium" />
<project name="platform/external/grpc-grpc" />
<project name="platform/external/grpc-grpc-go" />
<project name="platform/external/grpc-grpc-java" />
<project name="platform/external/grub" />
<project name="platform/external/gsoap" />
<project name="platform/external/gtest" />
<project name="platform/external/guava" />
<project name="platform/external/guice" />
<project name="platform/external/gwp_asan" />
<project name="platform/external/hamcrest" />
<project name="platform/external/harfbuzz" />
<project name="platform/external/harfbuzz_ng" />
<project name="platform/external/hcidump" />
<project name="platform/external/honggfuzz" />
<project name="platform/external/hsqldb" />
<project name="platform/external/hyphenation" />
<project name="platform/external/hyphenation-patterns" />
<project name="platform/external/icing" />
<project name="platform/external/icu" />
<project name="platform/external/icu4c" />
<project name="platform/external/id3lib" />
<project name="platform/external/igt-gpu-tools" />
<project name="platform/external/image_io" />
<project name="platform/external/ims" />
<project name="platform/external/iosched" />
<project name="platform/external/iperf3" />
<project name="platform/external/iproute2" />
<project name="platform/external/ipsec-tools" />
<project name="platform/external/iptables" />
<project name="platform/external/iputils" />
<project name="platform/external/iw" />
<project name="platform/external/jack" />
<project name="platform/external/jackson-annotations" />
<project name="platform/external/jackson-core" />
<project name="platform/external/jackson-databind" />
<project name="platform/external/jacoco" />
<project name="platform/external/jarjar" />
<project name="platform/external/javaparser" />
<project name="platform/external/javapoet" />
<project name="platform/external/javasqlite" />
<project name="platform/external/javassist" />
<project name="platform/external/jcommander" />
<project name="platform/external/jdiff" />
<project name="platform/external/jemalloc" />
<project name="platform/external/jemalloc_new" />
<project name="platform/external/jenkins-hash" />
<project name="platform/external/jetbrains/JetBrainsRuntime" />
<project name="platform/external/jetbrains/intellij-kotlin" />
<project name="platform/external/jetbrains/jdk8u" />
<project name="platform/external/jetbrains/jdk8u_corba" />
<project name="platform/external/jetbrains/jdk8u_hotspot" />
<project name="platform/external/jetbrains/jdk8u_jaxp" />
<project name="platform/external/jetbrains/jdk8u_jaxws" />
<project name="platform/external/jetbrains/jdk8u_jdk" />
<project name="platform/external/jetbrains/jdk8u_jfx" />
<project name="platform/external/jetbrains/jdk8u_langtools" />
<project name="platform/external/jetbrains/jdk8u_nashorn" />
<project name="platform/external/jetbrains/kotlin" />
<project name="platform/external/jetty" />
<project name="platform/external/jhead" />
<project name="platform/external/jimfs" />
<project name="platform/external/jline" />
<project name="platform/external/jmdns" />
<project name="platform/external/jmonkeyengine" />
<project name="platform/external/jpeg" />
<project name="platform/external/jsilver" />
<project name="platform/external/jsmn" />
<project name="platform/external/jsoncpp" />
<project name="platform/external/jsr305" />
<project name="platform/external/jsr330" />
<project name="platform/external/junit" />
<project name="platform/external/junit-params" />
<project name="platform/external/kernel-headers" />
<project name="platform/external/kmod" />
<project name="platform/external/kotlinc" />
<project name="platform/external/kotlinx.atomicfu" />
<project name="platform/external/kotlinx.coroutines" />
<project name="platform/external/kotlinx.metadata" />
<project name="platform/external/ksoap2" />
<project name="platform/external/ktlint" />
<project name="platform/external/kythe" />
<project name="platform/external/lame" />
<project name="platform/external/lcc" />
<project name="platform/external/libabigail" />
<project name="platform/external/libaom" />
<project name="platform/external/libavc" />
<project name="platform/external/libbackup" />
<project name="platform/external/libbpf" />
<project name="platform/external/libbrillo" />
<project name="platform/external/libc-test" />
<project name="platform/external/libcap" />
<project name="platform/external/libcap-ng" />
<project name="platform/external/libchrome" />
<project name="platform/external/libchromeos" />
<project name="platform/external/libchromeos-rs" />
<project name="platform/external/libconfig" />
<project name="platform/external/libconstrainedcrypto" />
<project name="platform/external/libcppbor" />
<project name="platform/external/libcups" />
<project name="platform/external/libcxx" />
<project name="platform/external/libcxx_35a" />
<project name="platform/external/libcxxabi" />
<project name="platform/external/libcxxabi_35a" />
<project name="platform/external/libcxxrt" />
<project name="platform/external/libdaemon" />
<project name="platform/external/libdivsufsort" />
<project name="platform/external/libdrm" />
<project name="platform/external/libedit" />
<project name="platform/external/libepoxy" />
<project name="platform/external/libese" />
<project name="platform/external/libevent" />
<project name="platform/external/libexif" />
<project name="platform/external/libffi" />
<project name="platform/external/libfuse" />
<project name="platform/external/libgav1" />
<project name="platform/external/libgdx" />
<project name="platform/external/libgsm" />
<project name="platform/external/libhevc" />
<project name="platform/external/libiio" />
<project name="platform/external/libiota" />
<project name="platform/external/libjpeg-turbo" />
<project name="platform/external/libkmsxx" />
<project name="platform/external/libldac" />
<project name="platform/external/liblzf" />
<project name="platform/external/libmicrohttpd" />
<project name="platform/external/libmojo" />
<project name="platform/external/libmpeg2" />
<project name="platform/external/libmtp" />
<project name="platform/external/libnetfilter_conntrack" />
<project name="platform/external/libnfc-nci" />
<project name="platform/external/libnfc-nxp" />
<project name="platform/external/libnfnetlink" />
<project name="platform/external/libnl" />
<project name="platform/external/libnl-headers" />
<project name="platform/external/libogg" />
<project name="platform/external/libopus" />
<project name="platform/external/libpcap" />
<project name="platform/external/libphonenumber" />
<project name="platform/external/libpng" />
<project name="platform/external/libppp" />
<project name="platform/external/libprotobuf-mutator" />
<project name="platform/external/libseccomp-helper" />
<project name="platform/external/libselinux" />
<project name="platform/external/libsepol" />
<project name="platform/external/libsrtp2" />
<project name="platform/external/libssh2" />
<project name="platform/external/libtextclassifier" />
<project name="platform/external/libunwind" />
<project name="platform/external/libunwind_llvm" />
<project name="platform/external/liburing" />
<project name="platform/external/libusb" />
<project name="platform/external/libusb-compat" />
<project name="platform/external/libusb_aah" />
<project name="platform/external/libutf" />
<project name="platform/external/libuv" />
<project name="platform/external/libvncserver" />
<project name="platform/external/libvorbis" />
<project name="platform/external/libvpx" />
<project name="platform/external/libvterm" />
<project name="platform/external/libweave" />
<project name="platform/external/libwebm" />
<project name="platform/external/libwebsockets" />
<project name="platform/external/libxaac" />
<project name="platform/external/libxcam" />
<project name="platform/external/libxkbcommon" />
<project name="platform/external/libxml2" />
<project name="platform/external/libxslt" />
<project name="platform/external/libyuv" />
<project name="platform/external/linux-kselftest" />
<project name="platform/external/linux-tools-perf" />
<project name="platform/external/lisa" />
<project name="platform/external/littlemock" />
<project name="platform/external/lld" />
<project name="platform/external/lldb" />
<project name="platform/external/lldb-utils" />
<project name="platform/external/llvm" />
<project name="platform/external/llvm_35a" />
<project name="platform/external/lmfit" />
<project name="platform/external/login-items-ae" />
<project name="platform/external/lohit-fonts" />
<project name="platform/external/lottie" />
<project name="platform/external/ltp" />
<project name="platform/external/ltrace" />
<project name="platform/external/lua" />
<project name="platform/external/lvm2" />
<project name="platform/external/lz4" />
<project name="platform/external/lzma" />
<project name="platform/external/lzop" />
<project name="platform/external/marisa-trie" />
<project name="platform/external/markdown" />
<project name="platform/external/mdnsresponder" />
<project name="platform/external/mesa3d" />
<project name="platform/external/messageformat" />
<project name="platform/external/mime-support" />
<project name="platform/external/minigbm" />
<project name="platform/external/minijail" />
<project name="platform/external/mksh" />
<project name="platform/external/mmc-utils" />
<project name="platform/external/mock" />
<project name="platform/external/mockftpserver" />
<project name="platform/external/mockito" />
<project name="platform/external/mockwebserver" />
<project name="platform/external/modp_b64" />
<project name="platform/external/mp4parser" />
<project name="platform/external/mp4v2" />
<project name="platform/external/mpg123" />
<project name="platform/external/ms-tpm-20-ref" />
<project name="platform/external/mtools" />
<project name="platform/external/mtpd" />
<project name="platform/external/musl" />
<project name="platform/external/nanohttpd" />
<project name="platform/external/nanopb-c" />
<project name="platform/external/nasm" />
<project name="platform/external/naver-fonts" />
<project name="platform/external/ncurses" />
<project name="platform/external/neon_2_sse" />
<project name="platform/external/netcat" />
<project name="platform/external/netperf" />
<project name="platform/external/neven" />
<project name="platform/external/newfs_msdos" />
<project name="platform/external/nfacct" />
<project name="platform/external/ninja" />
<project name="platform/external/nist-pkits" />
<project name="platform/external/nist-sip" />
<project name="platform/external/nos/host/android" />
<project name="platform/external/nos/host/generic" />
<project name="platform/external/nos/test/system-test-harness" />
<project name="platform/external/noto-fonts" />
<project name="platform/external/nsjail" />
<project name="platform/external/oauth" />
<project name="platform/external/objenesis" />
<project name="platform/external/oboe" />
<project name="platform/external/obstack" />
<project name="platform/external/oj-libjdwp" />
<project name="platform/external/okhttp" />
<project name="platform/external/okhttp4" />
<project name="platform/external/okio" />
<project name="platform/external/one-true-awk" />
<project name="platform/external/open-dice" />
<project name="platform/external/open-vcdiff" />
<project name="platform/external/opencensus-go" />
<project name="platform/external/opencensus-java" />
<project name="platform/external/opencore" />
<project name="platform/external/opencv" />
<project name="platform/external/opencv3" />
<project name="platform/external/openfst" />
<project name="platform/external/openmp_llvm" />
<project name="platform/external/openocd" />
<project name="platform/external/openscreen" />
<project name="platform/external/openssh" />
<project name="platform/external/openssl" />
<project name="platform/external/openthread" />
<project name="platform/external/openvpn" />
<project name="platform/external/openwrt-prebuilts" />
<project name="platform/external/oprofile" />
<project name="platform/external/optee/apps" />
<project name="platform/external/oss-fuzz" />
<project name="platform/external/owasp/sanitizer" />
<project name="platform/external/parameter-framework" />
<project name="platform/external/pcre" />
<project name="platform/external/pdfium" />
<project name="platform/external/perf_data_converter" />
<project name="platform/external/perfetto" />
<project name="platform/external/pffft" />
<project name="platform/external/piex" />
<project name="platform/external/pigweed" />
<project name="platform/external/ping" />
<project name="platform/external/ping6" />
<project name="platform/external/pixman" />
<project name="platform/external/ply" />
<project name="platform/external/ppp" />
<project name="platform/external/proguard" />
<project name="platform/external/protobuf" />
<project name="platform/external/protobuf-javalite" />
<project name="platform/external/protobuf2.5" />
<project name="platform/external/psimd" />
<project name="platform/external/pthreadpool" />
<project name="platform/external/pthreads" />
<project name="platform/external/puffin" />
<project name="platform/external/python" path="platform/external/python.git" />
<project name="platform/external/python/Pillow" />
<project name="platform/external/python/apitools" />
<project name="platform/external/python/appdirs" />
<project name="platform/external/python/asn1crypto" />
<project name="platform/external/python/atomicwrites" />
<project name="platform/external/python/attrs" />
<project name="platform/external/python/cachetools" />
<project name="platform/external/python/cffi" />
<project name="platform/external/python/cpython2" />
<project name="platform/external/python/cpython3" />
<project name="platform/external/python/cryptography" />
<project name="platform/external/python/dateutil" />
<project name="platform/external/python/dill" />
<project name="platform/external/python/enum" />
<project name="platform/external/python/enum34" />
<project name="platform/external/python/funcsigs" />
<project name="platform/external/python/future" />
<project name="platform/external/python/futures" />
<project name="platform/external/python/gapic-google-cloud-pubsub-v1" />
<project name="platform/external/python/google-api-python-client" />
<project name="platform/external/python/google-auth" />
<project name="platform/external/python/google-auth-httplib2" />
<project name="platform/external/python/google-cloud-core" />
<project name="platform/external/python/google-cloud-pubsub" />
<project name="platform/external/python/google-gax" />
<project name="platform/external/python/googleapis" />
<project name="platform/external/python/grpc-google-iam-v1" />
<project name="platform/external/python/grpcio" />
<project name="platform/external/python/httplib2" />
<project name="platform/external/python/ipaddress" />
<project name="platform/external/python/jinja" />
<project name="platform/external/python/markupsafe" />
<project name="platform/external/python/matplotlib" />
<project name="platform/external/python/mock" />
<project name="platform/external/python/more-itertools" />
<project name="platform/external/python/numpy" />
<project name="platform/external/python/oauth2client" />
<project name="platform/external/python/olefile" />
<project name="platform/external/python/packaging" />
<project name="platform/external/python/parse" />
<project name="platform/external/python/parse_type" />
<project name="platform/external/python/pluggy" />
<project name="platform/external/python/ply" />
<project name="platform/external/python/proto-google-cloud-pubsub-v1" />
<project name="platform/external/python/protobuf" />
<project name="platform/external/python/py" />
<project name="platform/external/python/pyasn1" />
<project name="platform/external/python/pyasn1-modules" />
<project name="platform/external/python/pybind11" />
<project name="platform/external/python/pycparser" />
<project name="platform/external/python/pyfakefs" />
<project name="platform/external/python/pyopenssl" />
<project name="platform/external/python/pyparsing" />
<project name="platform/external/python/pytest" />
<project name="platform/external/python/requests" />
<project name="platform/external/python/rsa" />
<project name="platform/external/python/scipy" />
<project name="platform/external/python/setuptools" />
<project name="platform/external/python/six" />
<project name="platform/external/python/uritemplates" />
<project name="platform/external/qemu" />
<project name="platform/external/qemu-android" />
<project name="platform/external/qemu-pc-bios" />
<project name="platform/external/qt" />
<project name="platform/external/quake" />
<project name="platform/external/r8" />
<project name="platform/external/rapidjson" />
<project name="platform/external/rappor" />
<project name="platform/external/regex-re2" />
<project name="platform/external/replicaisland" />
<project name="platform/external/rmi4utils" />
<project name="platform/external/rnnoise" />
<project name="platform/external/robolectric" />
<project name="platform/external/robolectric-shadows" />
<project name="platform/external/roboto-fonts" />
<project name="platform/external/rootdev" />
<project name="platform/external/rust/crates/ahash" />
<project name="platform/external/rust/crates/aho-corasick" />
<project name="platform/external/rust/crates/android_log-sys" />
<project name="platform/external/rust/crates/android_logger" />
<project name="platform/external/rust/crates/anyhow" />
<project name="platform/external/rust/crates/arbitrary" />
<project name="platform/external/rust/crates/ash" />
<project name="platform/external/rust/crates/async-stream" />
<project name="platform/external/rust/crates/async-stream-impl" />
<project name="platform/external/rust/crates/async-task" />
<project name="platform/external/rust/crates/async-trait" />
<project name="platform/external/rust/crates/atty" />
<project name="platform/external/rust/crates/base64" />
<project name="platform/external/rust/crates/bencher" />
<project name="platform/external/rust/crates/bincode" />
<project name="platform/external/rust/crates/bindgen" />
<project name="platform/external/rust/crates/bitflags" />
<project name="platform/external/rust/crates/bstr" />
<project name="platform/external/rust/crates/byteorder" />
<project name="platform/external/rust/crates/bytes" />
<project name="platform/external/rust/crates/bzip2" />
<project name="platform/external/rust/crates/bzip2-sys" />
<project name="platform/external/rust/crates/cast" />
<project name="platform/external/rust/crates/cesu8" />
<project name="platform/external/rust/crates/cexpr" />
<project name="platform/external/rust/crates/cfg-if" />
<project name="platform/external/rust/crates/chrono" />
<project name="platform/external/rust/crates/clang-sys" />
<project name="platform/external/rust/crates/clap" />
<project name="platform/external/rust/crates/codespan-reporting" />
<project name="platform/external/rust/crates/combine" />
<project name="platform/external/rust/crates/command-fds" />
<project name="platform/external/rust/crates/const_fn" />
<project name="platform/external/rust/crates/crc32fast" />
<project name="platform/external/rust/crates/criterion" />
<project name="platform/external/rust/crates/criterion-plot" />
<project name="platform/external/rust/crates/crossbeam-channel" />
<project name="platform/external/rust/crates/crossbeam-deque" />
<project name="platform/external/rust/crates/crossbeam-epoch" />
<project name="platform/external/rust/crates/crossbeam-queue" />
<project name="platform/external/rust/crates/crossbeam-utils" />
<project name="platform/external/rust/crates/csv" />
<project name="platform/external/rust/crates/csv-core" />
<project name="platform/external/rust/crates/der-oid-macro" />
<project name="platform/external/rust/crates/der-parser" />
<project name="platform/external/rust/crates/derive_arbitrary" />
<project name="platform/external/rust/crates/downcast-rs" />
<project name="platform/external/rust/crates/either" />
<project name="platform/external/rust/crates/env_logger" />
<project name="platform/external/rust/crates/fallible-iterator" />
<project name="platform/external/rust/crates/fallible-streaming-iterator" />
<project name="platform/external/rust/crates/flate2" />
<project name="platform/external/rust/crates/fnv" />
<project name="platform/external/rust/crates/form_urlencoded" />
<project name="platform/external/rust/crates/futures" />
<project name="platform/external/rust/crates/futures-channel" />
<project name="platform/external/rust/crates/futures-core" />
<project name="platform/external/rust/crates/futures-executor" />
<project name="platform/external/rust/crates/futures-io" />
<project name="platform/external/rust/crates/futures-macro" />
<project name="platform/external/rust/crates/futures-sink" />
<project name="platform/external/rust/crates/futures-task" />
<project name="platform/external/rust/crates/futures-util" />
<project name="platform/external/rust/crates/gdbstub" />
<project name="platform/external/rust/crates/gdbstub_arch" />
<project name="platform/external/rust/crates/getrandom" />
<project name="platform/external/rust/crates/glob" />
<project name="platform/external/rust/crates/grpcio" />
<project name="platform/external/rust/crates/grpcio-compiler" />
<project name="platform/external/rust/crates/grpcio-sys" />
<project name="platform/external/rust/crates/half" />
<project name="platform/external/rust/crates/hashbrown" />
<project name="platform/external/rust/crates/hashlink" />
<project name="platform/external/rust/crates/heck" />
<project name="platform/external/rust/crates/idna" />
<project name="platform/external/rust/crates/instant" />
<project name="platform/external/rust/crates/intrusive-collections" />
<project name="platform/external/rust/crates/itertools" />
<project name="platform/external/rust/crates/itoa" />
<project name="platform/external/rust/crates/jni" />
<project name="platform/external/rust/crates/jni-sys" />
<project name="platform/external/rust/crates/kernlog" />
<project name="platform/external/rust/crates/lazy_static" />
<project name="platform/external/rust/crates/lazycell" />
<project name="platform/external/rust/crates/libc" />
<project name="platform/external/rust/crates/libfuzzer-sys" />
<project name="platform/external/rust/crates/libloading" />
<project name="platform/external/rust/crates/libm" />
<project name="platform/external/rust/crates/libsqlite3-sys" />
<project name="platform/external/rust/crates/libz-sys" />
<project name="platform/external/rust/crates/linked-hash-map" />
<project name="platform/external/rust/crates/lock_api" />
<project name="platform/external/rust/crates/log" />
<project name="platform/external/rust/crates/lru-cache" />
<project name="platform/external/rust/crates/macaddr" />
<project name="platform/external/rust/crates/managed" />
<project name="platform/external/rust/crates/matches" />
<project name="platform/external/rust/crates/memchr" />
<project name="platform/external/rust/crates/memoffset" />
<project name="platform/external/rust/crates/minimal-lexical" />
<project name="platform/external/rust/crates/mio" />
<project name="platform/external/rust/crates/nix" />
<project name="platform/external/rust/crates/no-panic" />
<project name="platform/external/rust/crates/nom" />
<project name="platform/external/rust/crates/num-bigint" />
<project name="platform/external/rust/crates/num-derive" />
<project name="platform/external/rust/crates/num-integer" />
<project name="platform/external/rust/crates/num-traits" />
<project name="platform/external/rust/crates/num_cpus" />
<project name="platform/external/rust/crates/oid-registry" />
<project name="platform/external/rust/crates/once_cell" />
<project name="platform/external/rust/crates/oorandom" />
<project name="platform/external/rust/crates/parking_lot" />
<project name="platform/external/rust/crates/parking_lot_core" />
<project name="platform/external/rust/crates/paste" />
<project name="platform/external/rust/crates/paste-impl" />
<project name="platform/external/rust/crates/peeking_take_while" />
<project name="platform/external/rust/crates/percent-encoding" />
<project name="platform/external/rust/crates/pin-project" />
<project name="platform/external/rust/crates/pin-project-internal" />
<project name="platform/external/rust/crates/pin-project-lite" />
<project name="platform/external/rust/crates/pin-utils" />
<project name="platform/external/rust/crates/plotters" />
<project name="platform/external/rust/crates/plotters-backend" />
<project name="platform/external/rust/crates/plotters-svg" />
<project name="platform/external/rust/crates/ppv-lite86" />
<project name="platform/external/rust/crates/proc-macro-error" />
<project name="platform/external/rust/crates/proc-macro-error-attr" />
<project name="platform/external/rust/crates/proc-macro-hack" />
<project name="platform/external/rust/crates/proc-macro-nested" />
<project name="platform/external/rust/crates/proc-macro2" />
<project name="platform/external/rust/crates/protobuf" />
<project name="platform/external/rust/crates/protobuf-codegen" />
<project name="platform/external/rust/crates/quiche" />
<project name="platform/external/rust/crates/quickcheck" />
<project name="platform/external/rust/crates/quote" />
<project name="platform/external/rust/crates/rand" />
<project name="platform/external/rust/crates/rand_chacha" />
<project name="platform/external/rust/crates/rand_core" />
<project name="platform/external/rust/crates/rand_xorshift" />
<project name="platform/external/rust/crates/rayon" />
<project name="platform/external/rust/crates/rayon-core" />
<project name="platform/external/rust/crates/regex" />
<project name="platform/external/rust/crates/regex-automata" />
<project name="platform/external/rust/crates/regex-syntax" />
<project name="platform/external/rust/crates/remain" />
<project name="platform/external/rust/crates/ring" />
<project name="platform/external/rust/crates/rusqlite" />
<project name="platform/external/rust/crates/rustc-demangle" />
<project name="platform/external/rust/crates/rustc-demangle-capi" />
<project name="platform/external/rust/crates/rustc-hash" />
<project name="platform/external/rust/crates/rusticata-macros" />
<project name="platform/external/rust/crates/rustversion" />
<project name="platform/external/rust/crates/ryu" />
<project name="platform/external/rust/crates/same-file" />
<project name="platform/external/rust/crates/scopeguard" />
<project name="platform/external/rust/crates/serde" />
<project name="platform/external/rust/crates/serde-xml-rs" />
<project name="platform/external/rust/crates/serde_cbor" />
<project name="platform/external/rust/crates/serde_derive" />
<project name="platform/external/rust/crates/serde_json" />
<project name="platform/external/rust/crates/serde_test" />
<project name="platform/external/rust/crates/shared_child" />
<project name="platform/external/rust/crates/shared_library" />
<project name="platform/external/rust/crates/shlex" />
<project name="platform/external/rust/crates/slab" />
<project name="platform/external/rust/crates/smallvec" />
<project name="platform/external/rust/crates/spin" />
<project name="platform/external/rust/crates/standback" />
<project name="platform/external/rust/crates/structopt" />
<project name="platform/external/rust/crates/structopt-derive" />
<project name="platform/external/rust/crates/syn" />
<project name="platform/external/rust/crates/syn-mid" />
<project name="platform/external/rust/crates/termcolor" />
<project name="platform/external/rust/crates/textwrap" />
<project name="platform/external/rust/crates/thiserror" />
<project name="platform/external/rust/crates/thiserror-impl" />
<project name="platform/external/rust/crates/thread_local" />
<project name="platform/external/rust/crates/time" />
<project name="platform/external/rust/crates/time-macros" />
<project name="platform/external/rust/crates/time-macros-impl" />
<project name="platform/external/rust/crates/tinytemplate" />
<project name="platform/external/rust/crates/tinyvec" />
<project name="platform/external/rust/crates/tinyvec_macros" />
<project name="platform/external/rust/crates/tokio" />
<project name="platform/external/rust/crates/tokio-macros" />
<project name="platform/external/rust/crates/tokio-stream" />
<project name="platform/external/rust/crates/tokio-test" />
<project name="platform/external/rust/crates/unicode-bidi" />
<project name="platform/external/rust/crates/unicode-normalization" />
<project name="platform/external/rust/crates/unicode-segmentation" />
<project name="platform/external/rust/crates/unicode-width" />
<project name="platform/external/rust/crates/unicode-xid" />
<project name="platform/external/rust/crates/untrusted" />
<project name="platform/external/rust/crates/url" />
<project name="platform/external/rust/crates/uuid" />
<project name="platform/external/rust/crates/vmm_vhost" />
<project name="platform/external/rust/crates/vsock" />
<project name="platform/external/rust/crates/vulkano" />
<project name="platform/external/rust/crates/walkdir" />
<project name="platform/external/rust/crates/weak-table" />
<project name="platform/external/rust/crates/webpki" />
<project name="platform/external/rust/crates/which" />
<project name="platform/external/rust/crates/x509-parser" />
<project name="platform/external/rust/crates/xml-rs" />
<project name="platform/external/rust/crates/zip" />
<project name="platform/external/rust/cxx" />
<project name="platform/external/ruy" />
<project name="platform/external/s2-geometry-library-java" />
<project name="platform/external/safe-iop" />
<project name="platform/external/scapy" />
<project name="platform/external/scrypt" />
<project name="platform/external/scudo" />
<project name="platform/external/sdl2" />
<project name="platform/external/sdl2_ttf" />
<project name="platform/external/seccomp-tests" />
<project name="platform/external/selinux" />
<project name="platform/external/sepolicy" />
<project name="platform/external/setupcompat" />
<project name="platform/external/setupdesign" />
<project name="platform/external/sfntly" />
<project name="platform/external/shaderc/glslang" />
<project name="platform/external/shaderc/shaderc" />
<project name="platform/external/shaderc/spirv-headers" />
<project name="platform/external/shaderc/spirv-tools" />
<project name="platform/external/shflags" />
<project name="platform/external/sil-fonts" />
<project name="platform/external/skia" />
<project name="platform/external/skqp" />
<project name="platform/external/sl4a" />
<project name="platform/external/slf4j" />
<project name="platform/external/smack" />
<project name="platform/external/smali" />
<project name="platform/external/smaratorg" />
<project name="platform/external/snakeyaml" />
<project name="platform/external/sonic" />
<project name="platform/external/sonivox" />
<project name="platform/external/speex" />
<project name="platform/external/spirv-llvm" />
<project name="platform/external/sqlite" />
<project name="platform/external/squashfs-tools" />
<project name="platform/external/srec" />
<project name="platform/external/srtp" />
<project name="platform/external/stardoc" />
<project name="platform/external/starlark-go" />
<project name="platform/external/stlport" />
<project name="platform/external/strace" />
<project name="platform/external/stressapptest" />
<project name="platform/external/subsampling-scale-image-view" />
<project name="platform/external/svox" />
<project name="platform/external/swiftshader" />
<project name="platform/external/swig" />
<project name="platform/external/syslinux" />
<project name="platform/external/syspatch" />
<project name="platform/external/syzkaller" />
<project name="platform/external/tagsoup" />
<project name="platform/external/tcpdump" />
<project name="platform/external/tensorflow" />
<project name="platform/external/tesseract" />
<project name="platform/external/testng" />
<project name="platform/external/tflite-support" />
<project name="platform/external/timezone-boundary-builder" />
<project name="platform/external/timezonepicker-support" />
<project name="platform/external/tinyalsa" />
<project name="platform/external/tinyalsa_new" />
<project name="platform/external/tinycompress" />
<project name="platform/external/tinyobjloader" />
<project name="platform/external/tinyxml" />
<project name="platform/external/tinyxml2" />
<project name="platform/external/tlsdate" />
<project name="platform/external/toolchain-utils" />
<project name="platform/external/toybox" />
<project name="platform/external/tpm2" />
<project name="platform/external/tpm2-tss" />
<project name="platform/external/trappy" />
<project name="platform/external/tremolo" />
<project name="platform/external/tremor" />
<project name="platform/external/turbine" />
<project name="platform/external/u-boot" />
<project name="platform/external/ukey2" />
<project name="platform/external/unicode" />
<project name="platform/external/universal-tween-engine" />
<project name="platform/external/usrsctp" />
<project name="platform/external/utf8proc" />
<project name="platform/external/v4l2_codec2" />
<project name="platform/external/v8" />
<project name="platform/external/valgrind" />
<project name="platform/external/vboot_reference" />
<project name="platform/external/virglrenderer" />
<project name="platform/external/vixl" />
<project name="platform/external/vm_tools/p9" />
<project name="platform/external/vogar" />
<project name="platform/external/volley" />
<project name="platform/external/vulkan-headers" />
<project name="platform/external/vulkan-tools" />
<project name="platform/external/vulkan-validation-layers" />
<project name="platform/external/walt" />
<project name="platform/external/wayland" />
<project name="platform/external/wayland-protocols" />
<project name="platform/external/weave-common" />
<project name="platform/external/webkit" />
<project name="platform/external/webp" />
<project name="platform/external/webrtc" />
<project name="platform/external/webrtc_legacy" />
<project name="platform/external/webview_support_interfaces" />
<project name="platform/external/wmediumd" />
<project name="platform/external/wpa_supplicant" />
<project name="platform/external/wpa_supplicant_6" />
<project name="platform/external/wpa_supplicant_8" />
<project name="platform/external/wpantund" />
<project name="platform/external/wuffs-mirror-release-c" />
<project name="platform/external/wycheproof" />
<project name="platform/external/x264" />
<project name="platform/external/xdelta3" />
<project name="platform/external/xerces-cpp" />
<project name="platform/external/xmlrpcpp" />
<project name="platform/external/xmlwriter" />
<project name="platform/external/xmp_toolkit" />
<project name="platform/external/xz-embedded" />
<project name="platform/external/xz-java" />
<project name="platform/external/yaffs2" />
<project name="platform/external/yapf" />
<project name="platform/external/zlib" />
<project name="platform/external/zopfli" />
<project name="platform/external/zstd" />
<project name="platform/external/zucchini" />
<project name="platform/external/zxing" />
<project name="platform/frameworks/av" />
<project name="platform/frameworks/base" />
<project name="platform/frameworks/compile/libbcc" />
<project name="platform/frameworks/compile/linkloader" />
<project name="platform/frameworks/compile/llvm-ndk-cc" />
<project name="platform/frameworks/compile/mclinker" />
<project name="platform/frameworks/compile/slang" />
<project name="platform/frameworks/data" />
<project name="platform/frameworks/data-binding" />
<project name="platform/frameworks/ex" />
<project name="platform/frameworks/hardware/interfaces" />
<project name="platform/frameworks/janktesthelper" />
<project name="platform/frameworks/layoutlib" />
<project name="platform/frameworks/libs/modules-utils" />
<project name="platform/frameworks/libs/native_bridge_support" />
<project name="platform/frameworks/libs/net" />
<project name="platform/frameworks/libs/service_entitlement" />
<project name="platform/frameworks/libs/systemui" />
<project name="platform/frameworks/media/libvideoeditor" />
<project name="platform/frameworks/mff" />
<project name="platform/frameworks/minikin" />
<project name="platform/frameworks/ml" />
<project name="platform/frameworks/multidex" />
<project name="platform/frameworks/native" />
<project name="platform/frameworks/opt/bitmap" />
<project name="platform/frameworks/opt/bluetooth" />
<project name="platform/frameworks/opt/calendar" />
<project name="platform/frameworks/opt/car/services" />
<project name="platform/frameworks/opt/car/setupwizard" />
<project name="platform/frameworks/opt/carddav" />
<project name="platform/frameworks/opt/chips" />
<project name="platform/frameworks/opt/colorpicker" />
<project name="platform/frameworks/opt/com.google.android" />
<project name="platform/frameworks/opt/com.google.android.googlelogin" />
<project name="platform/frameworks/opt/datetimepicker" />
<project name="platform/frameworks/opt/emoji" />
<project name="platform/frameworks/opt/gamedevicets" />
<project name="platform/frameworks/opt/gamesdk" />
<project name="platform/frameworks/opt/inputconnectioncommon" />
<project name="platform/frameworks/opt/inputmethodcommon" />
<project name="platform/frameworks/opt/localepicker" />
<project name="platform/frameworks/opt/mailcommon" />
<project name="platform/frameworks/opt/mms" />
<project name="platform/frameworks/opt/net/ethernet" />
<project name="platform/frameworks/opt/net/ike" />
<project name="platform/frameworks/opt/net/ims" />
<project name="platform/frameworks/opt/net/lowpan" />
<project name="platform/frameworks/opt/net/voip" />
<project name="platform/frameworks/opt/net/wifi" />
<project name="platform/frameworks/opt/photoviewer" />
<project name="platform/frameworks/opt/setupwizard" />
<project name="platform/frameworks/opt/sherpa" />
<project name="platform/frameworks/opt/telephony" />
<project name="platform/frameworks/opt/timezonepicker" />
<project name="platform/frameworks/opt/tv/tvsystem" />
<project name="platform/frameworks/opt/vcard" />
<project name="platform/frameworks/opt/widget" />
<project name="platform/frameworks/policies/base" />
<project name="platform/frameworks/proto_logging" />
<project name="platform/frameworks/rs" />
<project name="platform/frameworks/support" />
<project name="platform/frameworks/support-golden" />
<project name="platform/frameworks/testing" />
<project name="platform/frameworks/uiautomator" />
<project name="platform/frameworks/volley" />
<project name="platform/frameworks/webview" />
<project name="platform/frameworks/wilhelm" />
<project name="platform/gdk" />
<project name="platform/hardware/akm" />
<project name="platform/hardware/broadcom/libbt" />
<project name="platform/hardware/broadcom/wlan" />
<project name="platform/hardware/bsp/bootloader/intel/edison-u-boot" />
<project name="platform/hardware/bsp/bootloader/nxp/uboot-imx" />
<project name="platform/hardware/bsp/bootloader/rockchip/rk-u-boot" />
<project name="platform/hardware/bsp/broadcom" />
<project name="platform/hardware/bsp/freescale" />
<project name="platform/hardware/bsp/imagination" />
<project name="platform/hardware/bsp/intel" />
<project name="platform/hardware/bsp/kernel/common/v4.1" />
<project name="platform/hardware/bsp/kernel/common/v4.4" />
<project name="platform/hardware/bsp/kernel/freescale" path="platform/hardware/bsp/kernel/freescale.git" />
<project name="platform/hardware/bsp/kernel/freescale/picoimx-3.14" />
<project name="platform/hardware/bsp/kernel/imagination/v4.1" />
<project name="platform/hardware/bsp/kernel/intel" path="platform/hardware/bsp/kernel/intel.git" />
<project name="platform/hardware/bsp/kernel/intel/broxton-v4.4" />
<project name="platform/hardware/bsp/kernel/intel/edison-v3.10" />
<project name="platform/hardware/bsp/kernel/intel/edison-v4.4" />
<project name="platform/hardware/bsp/kernel/intel/minnowboard-v3.14" />
<project name="platform/hardware/bsp/kernel/marvell/pxa-v3.14" />
<project name="platform/hardware/bsp/kernel/mediatek/mt8516-v4.4" />
<project name="platform/hardware/bsp/kernel/nxp/imx-v4.1" />
<project name="platform/hardware/bsp/kernel/nxp/imx-v4.9" />
<project name="platform/hardware/bsp/kernel/qcom" path="platform/hardware/bsp/kernel/qcom.git" />
<project name="platform/hardware/bsp/kernel/qcom/qcom-msm-v3.10" />
<project name="platform/hardware/bsp/kernel/qcom/qcom-msm-v4.9" />
<project name="platform/hardware/bsp/kernel/qcom/qcom-msm8x09-v3.10" />
<project name="platform/hardware/bsp/kernel/qcom/qcom-msm8x53-v3.18" />
<project name="platform/hardware/bsp/kernel/raspberrypi/pi-v4.1" />
<project name="platform/hardware/bsp/kernel/raspberrypi/pi-v4.4" />
<project name="platform/hardware/bsp/kernel/rockchip" path="platform/hardware/bsp/kernel/rockchip.git" />
<project name="platform/hardware/bsp/kernel/rockchip/rk-v4.4" />
<project name="platform/hardware/bsp/qcom" />
<project name="platform/hardware/bsp/rockchip" />
<project name="platform/hardware/google" path="platform/hardware/google.git" />
<project name="platform/hardware/google/apf" />
<project name="platform/hardware/google/av" />
<project name="platform/hardware/google/camera" />
<project name="platform/hardware/google/easel" />
<project name="platform/hardware/google/gchips" />
<project name="platform/hardware/google/graphics/common" />
<project name="platform/hardware/google/graphics/gs101" />
<project name="platform/hardware/google/interfaces" />
<project name="platform/hardware/google/pixel" />
<project name="platform/hardware/google/pixel-sepolicy" />
<project name="platform/hardware/intel/audio_media" />
<project name="platform/hardware/intel/bootstub" />
<project name="platform/hardware/intel/common/bd_prov" />
<project name="platform/hardware/intel/common/libmix" />
<project name="platform/hardware/intel/common/libstagefrighthw" />
<project name="platform/hardware/intel/common/libva" />
<project name="platform/hardware/intel/common/libwsbm" />
<project name="platform/hardware/intel/common/omx-components" />
<project name="platform/hardware/intel/common/utils" />
<project name="platform/hardware/intel/common/wrs_omxil_core" />
<project name="platform/hardware/intel/img/hwcomposer" />
<project name="platform/hardware/intel/img/libdrm" />
<project name="platform/hardware/intel/img/psb_headers" />
<project name="platform/hardware/intel/img/psb_video" />
<project name="platform/hardware/intel/sensors" />
<project name="platform/hardware/interfaces" />
<project name="platform/hardware/invensense" />
<project name="platform/hardware/knowles/athletico/sound_trigger_hal" />
<project name="platform/hardware/libhardware" />
<project name="platform/hardware/libhardware_legacy" />
<project name="platform/hardware/marvell/bt" />
<project name="platform/hardware/mediatek" />
<project name="platform/hardware/msm7k" />
<project name="platform/hardware/nvidia/audio" />
<project name="platform/hardware/nvidia/tegra124" />
<project name="platform/hardware/nxp/nfc" />
<project name="platform/hardware/nxp/secure_element" />
<project name="platform/hardware/qcom/audio" />
<project name="platform/hardware/qcom/bootctrl" />
<project name="platform/hardware/qcom/bt" />
<project name="platform/hardware/qcom/camera" />
<project name="platform/hardware/qcom/data/ipacfg-mgr" />
<project name="platform/hardware/qcom/display" />
<project name="platform/hardware/qcom/gps" />
<project name="platform/hardware/qcom/keymaster" />
<project name="platform/hardware/qcom/media" />
<project name="platform/hardware/qcom/msm8960" />
<project name="platform/hardware/qcom/msm8994" />
<project name="platform/hardware/qcom/msm8996" />
<project name="platform/hardware/qcom/msm8998" />
<project name="platform/hardware/qcom/msm8x09" />
<project name="platform/hardware/qcom/msm8x26" />
<project name="platform/hardware/qcom/msm8x27" />
<project name="platform/hardware/qcom/msm8x74" />
<project name="platform/hardware/qcom/msm8x84" />
<project name="platform/hardware/qcom/neuralnetworks/hvxservice" />
<project name="platform/hardware/qcom/power" />
<project name="platform/hardware/qcom/sdm710/data/ipacfg-mgr" />
<project name="platform/hardware/qcom/sdm710/display" />
<project name="platform/hardware/qcom/sdm710/gps" />
<project name="platform/hardware/qcom/sdm710/media" />
<project name="platform/hardware/qcom/sdm710/thermal" />
<project name="platform/hardware/qcom/sdm710/vr" />
<project name="platform/hardware/qcom/sdm845/bt" />
<project name="platform/hardware/qcom/sdm845/data/ipacfg-mgr" />
<project name="platform/hardware/qcom/sdm845/display" />
<project name="platform/hardware/qcom/sdm845/gps" />
<project name="platform/hardware/qcom/sdm845/media" />
<project name="platform/hardware/qcom/sdm845/thermal" />
<project name="platform/hardware/qcom/sdm845/vr" />
<project name="platform/hardware/qcom/sensors" />
<project name="platform/hardware/qcom/sm7150/display" />
<project name="platform/hardware/qcom/sm7150/gps" />
<project name="platform/hardware/qcom/sm7150/media" />
<project name="platform/hardware/qcom/sm7150/vr" />
<project name="platform/hardware/qcom/sm7250/display" />
<project name="platform/hardware/qcom/sm7250/gps" />
<project name="platform/hardware/qcom/sm7250/media" />
<project name="platform/hardware/qcom/sm8150/data/ipacfg-mgr" />
<project name="platform/hardware/qcom/sm8150/display" />
<project name="platform/hardware/qcom/sm8150/gps" />
<project name="platform/hardware/qcom/sm8150/media" />
<project name="platform/hardware/qcom/sm8150/thermal" />
<project name="platform/hardware/qcom/sm8150/vr" />
<project name="platform/hardware/qcom/sm8150p/gps" />
<project name="platform/hardware/qcom/wlan" />
<project name="platform/hardware/ril" />
<project name="platform/hardware/samsung/nfc" />
<project name="platform/hardware/samsung_slsi/exynos5" />
<project name="platform/hardware/st/nfc" />
<project name="platform/hardware/st/secure_element" />
<project name="platform/hardware/st/secure_element2" />
<project name="platform/hardware/telink/atv/refDesignRcu" />
<project name="platform/hardware/ti/am57x" />
<project name="platform/hardware/ti/omap3" />
<project name="platform/hardware/ti/omap4-aah" />
<project name="platform/hardware/ti/omap4xxx" />
<project name="platform/hardware/ti/wlan" />
<project name="platform/hardware/ti/wpan" />
<project name="platform/libcore" />
<project name="platform/libcore-snapshot" />
<project name="platform/libcore2" />
<project name="platform/libnativehelper" />
<project name="platform/manifest" />
<project name="platform/media/cts" />
<project name="platform/motodev" />
<project name="platform/ndk" />
<project name="platform/packages/apps/AccountsAndSyncSettings" />
<project name="platform/packages/apps/AlarmClock" />
<project name="platform/packages/apps/BasicSmsReceiver" />
<project name="platform/packages/apps/Benchmark" />
<project name="platform/packages/apps/Bluetooth" />
<project name="platform/packages/apps/Browser" />
<project name="platform/packages/apps/Browser2" />
<project name="platform/packages/apps/Calculator" />
<project name="platform/packages/apps/Calendar" />
<project name="platform/packages/apps/Camera" />
<project name="platform/packages/apps/Camera2" />
<project name="platform/packages/apps/Car/Calendar" />
<project name="platform/packages/apps/Car/Cluster" />
<project name="platform/packages/apps/Car/CompanionDeviceSupport" />
<project name="platform/packages/apps/Car/DebuggingRestrictionController" />
<project name="platform/packages/apps/Car/Dialer" />
<project name="platform/packages/apps/Car/Hvac" />
<project name="platform/packages/apps/Car/LatinIME" />
<project name="platform/packages/apps/Car/Launcher" />
<project name="platform/packages/apps/Car/LensPicker" />
<project name="platform/packages/apps/Car/LinkViewer" />
<project name="platform/packages/apps/Car/LocalMediaPlayer" />
<project name="platform/packages/apps/Car/Media" />
<project name="platform/packages/apps/Car/Messenger" />
<project name="platform/packages/apps/Car/Notification" />
<project name="platform/packages/apps/Car/Overview" />
<project name="platform/packages/apps/Car/Provision" />
<project name="platform/packages/apps/Car/Radio" />
<project name="platform/packages/apps/Car/RotaryController" />
<project name="platform/packages/apps/Car/Settings" />
<project name="platform/packages/apps/Car/SettingsIntelligence" />
<project name="platform/packages/apps/Car/Stream" />
<project name="platform/packages/apps/Car/SystemUI" />
<project name="platform/packages/apps/Car/SystemUpdater" />
<project name="platform/packages/apps/Car/Templates" />
<project name="platform/packages/apps/Car/UserManagement" />
<project name="platform/packages/apps/Car/externallibs" />
<project name="platform/packages/apps/Car/libs" />
<project name="platform/packages/apps/Car/tests" />
<project name="platform/packages/apps/CarrierConfig" />
<project name="platform/packages/apps/CellBroadcastReceiver" />
<project name="platform/packages/apps/CertInstaller" />
<project name="platform/packages/apps/Contacts" />
<project name="platform/packages/apps/ContactsCommon" />
<project name="platform/packages/apps/DeskClock" />
<project name="platform/packages/apps/DevCamera" />
<project name="platform/packages/apps/Dialer" />
<project name="platform/packages/apps/DocumentsUI" />
<project name="platform/packages/apps/Email" />
<project name="platform/packages/apps/EmergencyInfo" />
<project name="platform/packages/apps/ExactCalculator" />
<project name="platform/packages/apps/Exchange" />
<project name="platform/packages/apps/FMRadio" />
<project name="platform/packages/apps/Gallery" />
<project name="platform/packages/apps/Gallery2" />
<project name="platform/packages/apps/Gallery3D" />
<project name="platform/packages/apps/GlobalSearch" />
<project name="platform/packages/apps/GoogleSearch" />
<project name="platform/packages/apps/HTMLViewer" />
<project name="platform/packages/apps/IM" />
<project name="platform/packages/apps/IdentityCredentialSupport" />
<project name="platform/packages/apps/ImsServiceEntitlement" />
<project name="platform/packages/apps/InCallUI" />
<project name="platform/packages/apps/KeyChain" />
<project name="platform/packages/apps/Launcher" />
<project name="platform/packages/apps/Launcher2" />
<project name="platform/packages/apps/Launcher3" />
<project name="platform/packages/apps/LegacyCamera" />
<project name="platform/packages/apps/ManagedProvisioning" />
<project name="platform/packages/apps/Messaging" />
<project name="platform/packages/apps/Mms" />
<project name="platform/packages/apps/Music" />
<project name="platform/packages/apps/MusicFX" />
<project name="platform/packages/apps/Nfc" />
<project name="platform/packages/apps/OnDeviceAppPrediction" />
<project name="platform/packages/apps/OneTimeInitializer" />
<project name="platform/packages/apps/PackageInstaller" />
<project name="platform/packages/apps/Phone" />
<project name="platform/packages/apps/PhoneCommon" />
<project name="platform/packages/apps/Protips" />
<project name="platform/packages/apps/Provision" />
<project name="platform/packages/apps/QuickAccessWallet" />
<project name="platform/packages/apps/QuickSearchBox" />
<project name="platform/packages/apps/RemoteProvisioner" />
<project name="platform/packages/apps/RetailDemo" />
<project name="platform/packages/apps/SafetyRegulatoryInfo" />
<project name="platform/packages/apps/SampleLocationAttribution" />
<project name="platform/packages/apps/SecureElement" />
<project name="platform/packages/apps/Settings" />
<project name="platform/packages/apps/SettingsIntelligence" />
<project name="platform/packages/apps/SmartCardService" />
<project name="platform/packages/apps/SoundRecorder" />
<project name="platform/packages/apps/SpareParts" />
<project name="platform/packages/apps/SpeechRecorder" />
<project name="platform/packages/apps/Stk" />
<project name="platform/packages/apps/StorageManager" />
<project name="platform/packages/apps/Sync" />
<project name="platform/packages/apps/TV" />
<project name="platform/packages/apps/Tag" />
<project name="platform/packages/apps/Terminal" />
<project name="platform/packages/apps/Test/connectivity" />
<project name="platform/packages/apps/ThemePicker" />
<project name="platform/packages/apps/TimeZoneData" />
<project name="platform/packages/apps/TimeZoneUpdater" />
<project name="platform/packages/apps/Traceur" />
<project name="platform/packages/apps/TvSettings" />
<project name="platform/packages/apps/UnifiedEmail" />
<project name="platform/packages/apps/UniversalMediaPlayer" />
<project name="platform/packages/apps/Updater" />
<project name="platform/packages/apps/VideoEditor" />
<project name="platform/packages/apps/VoiceDialer" />
<project name="platform/packages/apps/WallpaperPicker" />
<project name="platform/packages/apps/WallpaperPicker2" />
<project name="platform/packages/experimental" />
<project name="platform/packages/inputmethods/LatinIME" />
<project name="platform/packages/inputmethods/LeanbackIME" />
<project name="platform/packages/inputmethods/OpenWnn" />
<project name="platform/packages/inputmethods/PinyinIME" />
<project name="platform/packages/modules/ANGLE" />
<project name="platform/packages/modules/ApiExtensions" />
<project name="platform/packages/modules/AppSearch" />
<project name="platform/packages/modules/ArtPrebuilt" />
<project name="platform/packages/modules/Bluetooth" />
<project name="platform/packages/modules/BootPrebuilt/5.10/arm64" />
<project name="platform/packages/modules/BootPrebuilt/5.4/arm64" />
<project name="platform/packages/modules/CaptivePortalLogin" />
<project name="platform/packages/modules/CellBroadcastService" />
<project name="platform/packages/modules/Connectivity" />
<project name="platform/packages/modules/Cronet" />
<project name="platform/packages/modules/DnsResolver" />
<project name="platform/packages/modules/ExtServices" />
<project name="platform/packages/modules/GeoTZ" />
<project name="platform/packages/modules/Gki" />
<project name="platform/packages/modules/IPsec" />
<project name="platform/packages/modules/Media" />
<project name="platform/packages/modules/MediaSwCodec" />
<project name="platform/packages/modules/ModuleMetadata" />
<project name="platform/packages/modules/NetworkPermissionConfig" />
<project name="platform/packages/modules/NetworkStack" />
<project name="platform/packages/modules/NeuralNetworks" />
<project name="platform/packages/modules/Permission" />
<project name="platform/packages/modules/PermissionController" />
<project name="platform/packages/modules/RuntimeI18n" />
<project name="platform/packages/modules/SEPolicy" />
<project name="platform/packages/modules/Scheduling" />
<project name="platform/packages/modules/SdkExtensions" />
<project name="platform/packages/modules/Sharesheet" />
<project name="platform/packages/modules/StatsD" />
<project name="platform/packages/modules/TestModule" />
<project name="platform/packages/modules/Tethering" />
<project name="platform/packages/modules/Virtualization" />
<project name="platform/packages/modules/Wifi" />
<project name="platform/packages/modules/adb" />
<project name="platform/packages/modules/common" />
<project name="platform/packages/modules/vndk" />
<project name="platform/packages/providers/ApplicationsProvider" />
<project name="platform/packages/providers/BlockedNumberProvider" />
<project name="platform/packages/providers/BookmarkProvider" />
<project name="platform/packages/providers/CalendarProvider" />
<project name="platform/packages/providers/CallLogProvider" />
<project name="platform/packages/providers/ContactsProvider" />
<project name="platform/packages/providers/DownloadProvider" />
<project name="platform/packages/providers/DrmProvider" />
<project name="platform/packages/providers/GoogleContactsProvider" />
<project name="platform/packages/providers/GoogleSubscribedFeedsProvider" />
<project name="platform/packages/providers/ImProvider" />
<project name="platform/packages/providers/ManagementProvider" />
<project name="platform/packages/providers/MediaProvider" />
<project name="platform/packages/providers/PartnerBookmarksProvider" />
<project name="platform/packages/providers/TelephonyProvider" />
<project name="platform/packages/providers/TvProvider" />
<project name="platform/packages/providers/UserDictionaryProvider" />
<project name="platform/packages/providers/WebSearchProvider" />
<project name="platform/packages/screensavers/Basic" />
<project name="platform/packages/screensavers/PhotoTable" />
<project name="platform/packages/screensavers/WebView" />
<project name="platform/packages/services/AlternativeNetworkAccess" />
<project name="platform/packages/services/BuiltInPrintService" />
<project name="platform/packages/services/Car" />
<project name="platform/packages/services/EasService" />
<project name="platform/packages/services/Iwlan" />
<project name="platform/packages/services/LockAndWipe" />
<project name="platform/packages/services/Mms" />
<project name="platform/packages/services/Mtp" />
<project name="platform/packages/services/NetworkRecommendation" />
<project name="platform/packages/services/Telecomm" />
<project name="platform/packages/services/Telephony" />
<project name="platform/packages/wallpapers/Basic" />
<project name="platform/packages/wallpapers/Galaxy4" />
<project name="platform/packages/wallpapers/HoloSpiral" />
<project name="platform/packages/wallpapers/ImageWallpaper" />
<project name="platform/packages/wallpapers/LivePicker" />
<project name="platform/packages/wallpapers/MagicSmoke" />
<project name="platform/packages/wallpapers/MusicVisualization" />
<project name="platform/packages/wallpapers/NoiseField" />
<project name="platform/packages/wallpapers/PhaseBeam" />
<project name="platform/pdk" />
<project name="platform/platform_testing" />
<project name="platform/prebuilt" />
<project name="platform/prebuilts/abi-dumps/ndk" />
<project name="platform/prebuilts/abi-dumps/platform" />
<project name="platform/prebuilts/abi-dumps/vndk" />
<project name="platform/prebuilts/android-emulator" />
<project name="platform/prebuilts/android-emulator-build/archive" />
<project name="platform/prebuilts/android-emulator-build/common" />
<project name="platform/prebuilts/android-emulator-build/curl" />
<project name="platform/prebuilts/android-emulator-build/mesa" />
<project name="platform/prebuilts/android-emulator-build/mesa-deps" />
<project name="platform/prebuilts/android-emulator-build/protobuf" />
<project name="platform/prebuilts/android-emulator-build/qemu-android-deps" />
<project name="platform/prebuilts/android-emulator-build/qt" />
<project name="platform/prebuilts/androidx/exoplayer" />
<project name="platform/prebuilts/androidx/external" />
<project name="platform/prebuilts/androidx/internal" />
<project name="platform/prebuilts/androidx/studio" />
<project name="platform/prebuilts/androidx/traceprocessor" />
<project name="platform/prebuilts/asuite" />
<project name="platform/prebuilts/au-generator" />
<project name="platform/prebuilts/bazel/darwin-x86_64" />
<project name="platform/prebuilts/bazel/linux-x86_64" />
<project name="platform/prebuilts/boot-artifacts" />
<project name="platform/prebuilts/build-artifacts" />
<project name="platform/prebuilts/build-artifacts2" />
<project name="platform/prebuilts/build-artifacts3" />
<project name="platform/prebuilts/build-tools" />
<project name="platform/prebuilts/bundletool" />
<project name="platform/prebuilts/checkcolor" />
<project name="platform/prebuilts/checkstyle" />
<project name="platform/prebuilts/clack" />
<project name="platform/prebuilts/clang-tools" />
<project name="platform/prebuilts/clang/darwin-x86/3.1" />
<project name="platform/prebuilts/clang/darwin-x86/3.2" />
<project name="platform/prebuilts/clang/darwin-x86/arm/3.3" />
<project name="platform/prebuilts/clang/darwin-x86/host/3.3" />
<project name="platform/prebuilts/clang/darwin-x86/host/3.4" />
<project name="platform/prebuilts/clang/darwin-x86/host/3.5" />
<project name="platform/prebuilts/clang/darwin-x86/host/3.6" />
<project name="platform/prebuilts/clang/darwin-x86/mips/3.3" />
<project name="platform/prebuilts/clang/darwin-x86/sdk/3.5" />
<project name="platform/prebuilts/clang/darwin-x86/x86/3.3" />
<project name="platform/prebuilts/clang/host/darwin-universal" />
<project name="platform/prebuilts/clang/host/darwin-x86" />
<project name="platform/prebuilts/clang/host/linux-x86" />
<project name="platform/prebuilts/clang/host/windows-x86" />
<project name="platform/prebuilts/clang/host/windows-x86_32" />
<project name="platform/prebuilts/clang/linux-x86/3.1" />
<project name="platform/prebuilts/clang/linux-x86/3.2" />
<project name="platform/prebuilts/clang/linux-x86/arm/3.3" />
<project name="platform/prebuilts/clang/linux-x86/host/3.3" />
<project name="platform/prebuilts/clang/linux-x86/host/3.4" />
<project name="platform/prebuilts/clang/linux-x86/host/3.5" />
<project name="platform/prebuilts/clang/linux-x86/host/3.6" />
<project name="platform/prebuilts/clang/linux-x86/mips/3.3" />
<project name="platform/prebuilts/clang/linux-x86/x86/3.3" />
<project name="platform/prebuilts/cmake/darwin-x86" />
<project name="platform/prebuilts/cmake/linux-x86" />
<project name="platform/prebuilts/cmake/windows-x86" />
<project name="platform/prebuilts/cmdline-tools" />
<project name="platform/prebuilts/deqp" />
<project name="platform/prebuilts/devtools" />
<project name="platform/prebuilts/eclipse" />
<project name="platform/prebuilts/eclipse-build-deps" />
<project name="platform/prebuilts/eclipse-build-deps-sources" />
<project name="platform/prebuilts/fuchsia_sdk" />
<project name="platform/prebuilts/fullsdk-darwin/build-tools/27.0.3" />
<project name="platform/prebuilts/fullsdk-darwin/build-tools/28.0.2" />
<project name="platform/prebuilts/fullsdk-darwin/build-tools/28.0.3" />
<project name="platform/prebuilts/fullsdk-darwin/build-tools/29.0.0" />
<project name="platform/prebuilts/fullsdk-darwin/build-tools/29.0.3" />
<project name="platform/prebuilts/fullsdk-darwin/build-tools/30.0.2" />
<project name="platform/prebuilts/fullsdk-darwin/build-tools/30.0.3" />
<project name="platform/prebuilts/fullsdk-darwin/build-tools/31.0.0" />
<project name="platform/prebuilts/fullsdk-darwin/platform-tools" />
<project name="platform/prebuilts/fullsdk-darwin/tools" />
<project name="platform/prebuilts/fullsdk-linux/build-tools/26.0.1" />
<project name="platform/prebuilts/fullsdk-linux/build-tools/27.0.3" />
<project name="platform/prebuilts/fullsdk-linux/build-tools/28.0.2" />
<project name="platform/prebuilts/fullsdk-linux/build-tools/28.0.3" />
<project name="platform/prebuilts/fullsdk-linux/build-tools/29.0.0" />
<project name="platform/prebuilts/fullsdk-linux/build-tools/29.0.3" />
<project name="platform/prebuilts/fullsdk-linux/build-tools/30.0.2" />
<project name="platform/prebuilts/fullsdk-linux/build-tools/30.0.3" />
<project name="platform/prebuilts/fullsdk-linux/build-tools/31.0.0" />
<project name="platform/prebuilts/fullsdk-linux/platform-tools" />
<project name="platform/prebuilts/fullsdk-linux/tools" />
<project name="platform/prebuilts/fullsdk/build-tools/21-darwin" />
<project name="platform/prebuilts/fullsdk/build-tools/21-linux" />
<project name="platform/prebuilts/fullsdk/build-tools/22-darwin" />
<project name="platform/prebuilts/fullsdk/build-tools/22-linux" />
<project name="platform/prebuilts/fullsdk/build-tools/22-windows" />
<project name="platform/prebuilts/fullsdk/platforms/android-21" />
<project name="platform/prebuilts/fullsdk/platforms/android-26" />
<project name="platform/prebuilts/fullsdk/platforms/android-28" />
<project name="platform/prebuilts/fullsdk/platforms/android-29" />
<project name="platform/prebuilts/fullsdk/platforms/android-30" />
<project name="platform/prebuilts/fullsdk/platforms/android-31" />
<project name="platform/prebuilts/fullsdk/platforms/android-system-29" />
<project name="platform/prebuilts/fullsdk/sources/android-28" />
<project name="platform/prebuilts/fullsdk/sources/android-29" />
<project name="platform/prebuilts/fullsdk/sources/android-30" />
<project name="platform/prebuilts/fullsdk/sources/android-31" />
<project name="platform/prebuilts/gas/linux-x86" />
<project name="platform/prebuilts/gcc/darwin-x86/aarch64/aarch64-linux-android-4.8" />
<project name="platform/prebuilts/gcc/darwin-x86/aarch64/aarch64-linux-android-4.9" />
<project name="platform/prebuilts/gcc/darwin-x86/arm/arm-eabi-4.6" />
<project name="platform/prebuilts/gcc/darwin-x86/arm/arm-eabi-4.7" />
<project name="platform/prebuilts/gcc/darwin-x86/arm/arm-eabi-4.8" />
<project name="platform/prebuilts/gcc/darwin-x86/arm/arm-linux-androideabi-4.6" />
<project name="platform/prebuilts/gcc/darwin-x86/arm/arm-linux-androideabi-4.7" />
<project name="platform/prebuilts/gcc/darwin-x86/arm/arm-linux-androideabi-4.8" />
<project name="platform/prebuilts/gcc/darwin-x86/arm/arm-linux-androideabi-4.9" />
<project name="platform/prebuilts/gcc/darwin-x86/host/headers" />
<project name="platform/prebuilts/gcc/darwin-x86/host/i686-apple-darwin-4.2.1" />
<project name="platform/prebuilts/gcc/darwin-x86/mips/mips64el-linux-android-4.8" />
<project name="platform/prebuilts/gcc/darwin-x86/mips/mips64el-linux-android-4.9" />
<project name="platform/prebuilts/gcc/darwin-x86/mips/mipsel-linux-android-4.4.3" />
<project name="platform/prebuilts/gcc/darwin-x86/mips/mipsel-linux-android-4.6" />
<project name="platform/prebuilts/gcc/darwin-x86/mips/mipsel-linux-android-4.7" />
<project name="platform/prebuilts/gcc/darwin-x86/mips/mipsel-linux-android-4.8" />
<project name="platform/prebuilts/gcc/darwin-x86/x86/i686-android-linux-4.4.3" />
<project name="platform/prebuilts/gcc/darwin-x86/x86/i686-android-linux-4.6" />
<project name="platform/prebuilts/gcc/darwin-x86/x86/i686-linux-android-4.6" />
<project name="platform/prebuilts/gcc/darwin-x86/x86/i686-linux-android-4.7" />
<project name="platform/prebuilts/gcc/darwin-x86/x86/x86_64-linux-android-4.7" />
<project name="platform/prebuilts/gcc/darwin-x86/x86/x86_64-linux-android-4.8" />
<project name="platform/prebuilts/gcc/darwin-x86/x86/x86_64-linux-android-4.9" />
<project name="platform/prebuilts/gcc/linux-x86/aarch64/aarch64-linux-android-4.8" />
<project name="platform/prebuilts/gcc/linux-x86/aarch64/aarch64-linux-android-4.9" />
<project name="platform/prebuilts/gcc/linux-x86/arm/arm-eabi-4.6" />
<project name="platform/prebuilts/gcc/linux-x86/arm/arm-eabi-4.7" />
<project name="platform/prebuilts/gcc/linux-x86/arm/arm-eabi-4.8" />
<project name="platform/prebuilts/gcc/linux-x86/arm/arm-linux-androideabi-4.6" />
<project name="platform/prebuilts/gcc/linux-x86/arm/arm-linux-androideabi-4.7" />
<project name="platform/prebuilts/gcc/linux-x86/arm/arm-linux-androideabi-4.8" />
<project name="platform/prebuilts/gcc/linux-x86/arm/arm-linux-androideabi-4.9" />
<project name="platform/prebuilts/gcc/linux-x86/host/i686-linux-glibc2.7-4.4.3" />
<project name="platform/prebuilts/gcc/linux-x86/host/i686-linux-glibc2.7-4.6" />
<project name="platform/prebuilts/gcc/linux-x86/host/x86_64-linux-glibc2.11-4.6" />
<project name="platform/prebuilts/gcc/linux-x86/host/x86_64-linux-glibc2.11-4.8" />
<project name="platform/prebuilts/gcc/linux-x86/host/x86_64-linux-glibc2.15-4.8" />
<project name="platform/prebuilts/gcc/linux-x86/host/x86_64-linux-glibc2.17-4.8" />
<project name="platform/prebuilts/gcc/linux-x86/host/x86_64-linux-glibc2.19-4.9" />
<project name="platform/prebuilts/gcc/linux-x86/host/x86_64-linux-glibc2.7-4.6" />
<project name="platform/prebuilts/gcc/linux-x86/host/x86_64-w64-mingw32-4.8" />
<project name="platform/prebuilts/gcc/linux-x86/mips/mips64el-linux-android-4.8" />
<project name="platform/prebuilts/gcc/linux-x86/mips/mips64el-linux-android-4.9" />
<project name="platform/prebuilts/gcc/linux-x86/mips/mipsel-linux-android-4.4.3" />
<project name="platform/prebuilts/gcc/linux-x86/mips/mipsel-linux-android-4.6" />
<project name="platform/prebuilts/gcc/linux-x86/mips/mipsel-linux-android-4.7" />
<project name="platform/prebuilts/gcc/linux-x86/mips/mipsel-linux-android-4.8" />
<project name="platform/prebuilts/gcc/linux-x86/x86/i686-android-linux-4.4.3" />
<project name="platform/prebuilts/gcc/linux-x86/x86/i686-android-linux-4.6" />
<project name="platform/prebuilts/gcc/linux-x86/x86/i686-linux-android-4.6" />
<project name="platform/prebuilts/gcc/linux-x86/x86/i686-linux-android-4.7" />
<project name="platform/prebuilts/gcc/linux-x86/x86/x86_64-linux-android-4.7" />
<project name="platform/prebuilts/gcc/linux-x86/x86/x86_64-linux-android-4.8" />
<project name="platform/prebuilts/gcc/linux-x86/x86/x86_64-linux-android-4.9" />
<project name="platform/prebuilts/gdb/darwin-x86" />
<project name="platform/prebuilts/gdb/linux-x86" />
<project name="platform/prebuilts/go/darwin-x86" />
<project name="platform/prebuilts/go/linux-x86" />
<project name="platform/prebuilts/go/windows-x86" />
<project name="platform/prebuilts/google-breakpad/darwin-x86" />
<project name="platform/prebuilts/google-breakpad/linux-x86" />
<project name="platform/prebuilts/google-breakpad/windows-x86" />
<project name="platform/prebuilts/gradle-plugin" />
<project name="platform/prebuilts/jdk/jdk11" />
<project name="platform/prebuilts/jdk/jdk8" />
<project name="platform/prebuilts/jdk/jdk9" />
<project name="platform/prebuilts/ktlint" />
<project name="platform/prebuilts/libedit/darwin-x86" />
<project name="platform/prebuilts/libedit/linux-x86" />
<project name="platform/prebuilts/libglog/darwin-x86" />
<project name="platform/prebuilts/libglog/linux-x86" />
<project name="platform/prebuilts/libglog/windows-x86" />
<project name="platform/prebuilts/libprotobuf/darwin" />
<project name="platform/prebuilts/libprotobuf/darwin-x86" />
<project name="platform/prebuilts/libprotobuf/linux" />
<project name="platform/prebuilts/libprotobuf/linux-x86" />
<project name="platform/prebuilts/libprotobuf/windows" />
<project name="platform/prebuilts/libprotobuf/windows-x86" />
<project name="platform/prebuilts/libs/libedit" />
<project name="platform/prebuilts/manifest-merger" />
<project name="platform/prebuilts/maven_repo/android" />
<project name="platform/prebuilts/maven_repo/bumptech" />
<project name="platform/prebuilts/maven_repo/google-play-service-client-libraries-3p" />
<project name="platform/prebuilts/misc" />
<project name="platform/prebuilts/module_sdk/Connectivity" />
<project name="platform/prebuilts/module_sdk/IPsec" />
<project name="platform/prebuilts/module_sdk/Media" />
<project name="platform/prebuilts/module_sdk/MediaProvider" />
<project name="platform/prebuilts/module_sdk/Permission" />
<project name="platform/prebuilts/module_sdk/Scheduling" />
<project name="platform/prebuilts/module_sdk/SdkExtensions" />
<project name="platform/prebuilts/module_sdk/StatsD" />
<project name="platform/prebuilts/module_sdk/Wifi" />
<project name="platform/prebuilts/module_sdk/art" />
<project name="platform/prebuilts/module_sdk/conscrypt" />
<project name="platform/prebuilts/ndk" />
<project name="platform/prebuilts/ninja/darwin-x86" />
<project name="platform/prebuilts/ninja/linux-x86" />
<project name="platform/prebuilts/ninja/windows-x86" />
<project name="platform/prebuilts/protobuf" />
<project name="platform/prebuilts/python/darwin-x86" path="platform/prebuilts/python/darwin-x86.git" />
<project name="platform/prebuilts/python/darwin-x86/2.7.5" />
<project name="platform/prebuilts/python/linux-x86" path="platform/prebuilts/python/linux-x86.git" />
<project name="platform/prebuilts/python/linux-x86/2.7.5" />
<project name="platform/prebuilts/python/windows-x86" />
<project name="platform/prebuilts/qemu-kernel" />
<project name="platform/prebuilts/r8" />
<project name="platform/prebuilts/remoteexecution-client" />
<project name="platform/prebuilts/renderscript/host/darwin-x86" />
<project name="platform/prebuilts/renderscript/host/linux-x86" />
<project name="platform/prebuilts/renderscript/host/windows-x86" />
<project name="platform/prebuilts/runtime" />
<project name="platform/prebuilts/rust" />
<project name="platform/prebuilts/sdk" />
<project name="platform/prebuilts/simpleperf" />
<project name="platform/prebuilts/studio/jdk" />
<project name="platform/prebuilts/studio/layoutlib" />
<project name="platform/prebuilts/swig/darwin-x86" />
<project name="platform/prebuilts/swig/linux-x86" />
<project name="platform/prebuilts/swig/windows-x86" />
<project name="platform/prebuilts/tools" />
<project name="platform/prebuilts/vndk/v27" />
<project name="platform/prebuilts/vndk/v28" />
<project name="platform/prebuilts/vndk/v29" />
<project name="platform/prebuilts/vndk/v30" />
<project name="platform/prebuilts/vndk/v31" />
<project name="platform/sdk" />
<project name="platform/smaratorg" />
<project name="platform/superproject" />
<project name="platform/system/adb" />
<project name="platform/system/apex" />
<project name="platform/system/ashmemd" />
<project name="platform/system/attestation" />
<project name="platform/system/bluetooth" />
<project name="platform/system/bpf" />
<project name="platform/system/bpfprogs" />
<project name="platform/system/bt" />
<project name="platform/system/bvb" />
<project name="platform/system/ca-certificates" />
<project name="platform/system/chre" />
<project name="platform/system/connectivity/apmanager" />
<project name="platform/system/connectivity/dhcp_client" />
<project name="platform/system/connectivity/shill" />
<project name="platform/system/connectivity/wificond" />
<project name="platform/system/connectivity/wifilogd" />
<project name="platform/system/core" />
<project name="platform/system/crash_reporter" />
<project name="platform/system/extras" />
<project name="platform/system/firewalld" />
<project name="platform/system/gatekeeper" />
<project name="platform/system/gsid" />
<project name="platform/system/hardware/interfaces" />
<project name="platform/system/hwservicemanager" />
<project name="platform/system/incremental_delivery" />
<project name="platform/system/iorap" />
<project name="platform/system/iot/attestation" />
<project name="platform/system/iot/mtdutils" />
<project name="platform/system/iot/tools" />
<project name="platform/system/keyguard" />
<project name="platform/system/keymaster" />
<project name="platform/system/libartpalette" />
<project name="platform/system/libbase" />
<project name="platform/system/libfmq" />
<project name="platform/system/libhidl" />
<project name="platform/system/libhwbinder" />
<project name="platform/system/libprocinfo" />
<project name="platform/system/librustutils" />
<project name="platform/system/libsysprop" />
<project name="platform/system/libufdt" />
<project name="platform/system/libvintf" />
<project name="platform/system/libziparchive" />
<project name="platform/system/linkerconfig" />
<project name="platform/system/logging" />
<project name="platform/system/media" />
<project name="platform/system/memory/libdmabufheap" />
<project name="platform/system/memory/libion" />
<project name="platform/system/memory/libmeminfo" />
<project name="platform/system/memory/libmemtrack" />
<project name="platform/system/memory/libmemunreachable" />
<project name="platform/system/memory/lmkd" />
<project name="platform/system/memory/mm_eventsd" />
<project name="platform/system/metricsd" />
<project name="platform/system/nativepower" />
<project name="platform/system/netd" />
<project name="platform/system/nfc" />
<project name="platform/system/nvram" />
<project name="platform/system/peripheralmanager" />
<project name="platform/system/security" />
<project name="platform/system/sepolicy" />
<project name="platform/system/server_configurable_flags" />
<project name="platform/system/teeui" />
<project name="platform/system/testing/gtest_extras" />
<project name="platform/system/timezone" />
<project name="platform/system/tools/aidl" />
<project name="platform/system/tools/bpt" />
<project name="platform/system/tools/hidl" />
<project name="platform/system/tools/mkbootimg" />
<project name="platform/system/tools/sysprop" />
<project name="platform/system/tools/xsdc" />
<project name="platform/system/tpm" />
<project name="platform/system/tpm_manager" />
<project name="platform/system/trunks" />
<project name="platform/system/ucontainer" />
<project name="platform/system/unwinding" />
<project name="platform/system/update_engine" />
<project name="platform/system/vold" />
<project name="platform/system/weaved" />
<project name="platform/system/webservd" />
<project name="platform/system/wifi" />
<project name="platform/system/wlan/ti" />
<project name="platform/test/AfwTestHarness" />
<project name="platform/test/app_compat/csuite" />
<project name="platform/test/catbox" />
<project name="platform/test/cts-root" />
<project name="platform/test/dittosuite" />
<project name="platform/test/framework" />
<project name="platform/test/mlts/benchmark" />
<project name="platform/test/mlts/models" />
<project name="platform/test/mts" />
<project name="platform/test/p2pts" />
<project name="platform/test/sts" />
<project name="platform/test/suite_harness" />
<project name="platform/test/vti/alert" />
<project name="platform/test/vti/dashboard" />
<project name="platform/test/vti/fuzz_test_serving" />
<project name="platform/test/vti/test_serving" />
<project name="platform/test/vts" />
<project name="platform/test/vts-testcase/fuzz" />
<project name="platform/test/vts-testcase/hal" />
<project name="platform/test/vts-testcase/hal-trace" />
<project name="platform/test/vts-testcase/kernel" />
<project name="platform/test/vts-testcase/nbu" />
<project name="platform/test/vts-testcase/performance" />
<project name="platform/test/vts-testcase/security" />
<project name="platform/test/vts-testcase/vndk" />
<project name="platform/tools/aadevtools" />
<project name="platform/tools/acloud" />
<project name="platform/tools/adt/eclipse" />
<project name="platform/tools/adt/idea" />
<project name="platform/tools/analytics-library" />
<project name="platform/tools/apifinder" />
<project name="platform/tools/apksig" />
<project name="platform/tools/apkzlib" />
<project name="platform/tools/appbundle" />
<project name="platform/tools/asuite" />
<project name="platform/tools/base" />
<project name="platform/tools/bdk" />
<project name="platform/tools/build" />
<project name="platform/tools/buildSrc" />
<project name="platform/tools/carrier_settings" />
<project name="platform/tools/cmake-utils" />
<project name="platform/tools/currysrc" />
<project name="platform/tools/dctv-tracedb" />
<project name="platform/tools/dexter" />
<project name="platform/tools/doc_generation" />
<project name="platform/tools/dokka-devsite-plugin" />
<project name="platform/tools/emulator" />
<project name="platform/tools/external/bazelbuild-rules-kotlin" />
<project name="platform/tools/external/fat32lib" />
<project name="platform/tools/external/go/src/github.com/go-gl-legacy/gl" />
<project name="platform/tools/external/go/src/github.com/go-gl/glfw" />
<project name="platform/tools/external/go/src/github.com/golang/protobuf" />
<project name="platform/tools/external/go/src/golang.org/x/net" />
<project name="platform/tools/external/go/src/golang.org/x/tools" />
<project name="platform/tools/external/gradle" />
<project name="platform/tools/external_updater" />
<project name="platform/tools/google_prebuilts/studio/sdk/remote" />
<project name="platform/tools/gpu" />
<project name="platform/tools/gradle" />
<project name="platform/tools/gradle-recipes" />
<project name="platform/tools/idea" />
<project name="platform/tools/loganalysis" />
<project name="platform/tools/metalava" />
<project name="platform/tools/motodev" />
<project name="platform/tools/multitest_transport" />
<project name="platform/tools/ndkports" />
<project name="platform/tools/repohooks" />
<project name="platform/tools/repohooks-trampoline" />
<project name="platform/tools/security" />
<project name="platform/tools/studio/cloud" />
<project name="platform/tools/studio/google/appindexing" />
<project name="platform/tools/studio/google/cloud/testing" />
<project name="platform/tools/studio/google/cloud/tools" />
<project name="platform/tools/studio/google/login" />
<project name="platform/tools/studio/google/play" />
<project name="platform/tools/studio/google/samples" />
<project name="platform/tools/studio/google/services" />
<project name="platform/tools/studio/translation" />
<project name="platform/tools/swing-testing" />
<project name="platform/tools/swt" />
<project name="platform/tools/test/connectivity" />
<project name="platform/tools/test/graphicsbenchmark" />
<project name="platform/tools/test/openhst" />
<project name="platform/tools/tradefed_cluster" />
<project name="platform/tools/tradefederation" path="platform/tools/tradefederation.git" />
<project name="platform/tools/tradefederation/contrib" />
<project name="platform/tools/tradefederation/prebuilts" />
<project name="platform/tools/treble" />
<project name="platform/tools/trebuchet" />
<project name="platform/vendor/htc/dream-open" />
<project name="platform/vendor/invensense" />
<project name="platform/vendor/sample" />
<project name="product/google/common" />
<project name="product/google/example-ledflasher" />
<project name="toolchain/android_rust" />
<project name="toolchain/avr-libc" />
<project name="toolchain/benchmark" />
<project name="toolchain/binutils" />
<project name="toolchain/build" />
<project name="toolchain/ccache" />
<project name="toolchain/clang" />
<project name="toolchain/clang-tools-extra" />
<project name="toolchain/cloog" />
<project name="toolchain/compiler-rt" />
<project name="toolchain/expat" />
<project name="toolchain/gcc" />
<project name="toolchain/gdb" />
<project name="toolchain/gmp" />
<project name="toolchain/gold" />
<project name="toolchain/isl" />
<project name="toolchain/jack" />
<project name="toolchain/jack-server" />
<project name="toolchain/jdk/build" />
<project name="toolchain/jdk/jdk11" />
<project name="toolchain/jdk/jdk9" />
<project name="toolchain/jdk/jdk9_corba" />
<project name="toolchain/jdk/jdk9_hotspot" />
<project name="toolchain/jdk/jdk9_jaxp" />
<project name="toolchain/jdk/jdk9_jaxws" />
<project name="toolchain/jdk/jdk9_jdk" />
<project name="toolchain/jdk/jdk9_langtools" />
<project name="toolchain/jdk/jdk9_nashorn" />
<project name="toolchain/jill" />
<project name="toolchain/libcxx" />
<project name="toolchain/libcxxabi" />
<project name="toolchain/lld" />
<project name="toolchain/llvm" />
<project name="toolchain/llvm-project" />
<project name="toolchain/llvm_android" />
<project name="toolchain/m4" />
<project name="toolchain/make" />
<project name="toolchain/manifest" />
<project name="toolchain/mclinker" />
<project name="toolchain/mingw" />
<project name="toolchain/mpc" />
<project name="toolchain/mpfr" />
<project name="toolchain/ndk-kokoro" />
<project name="toolchain/ndk_chromite_config" />
<project name="toolchain/openmp_llvm" />
<project name="toolchain/perl" />
<project name="toolchain/pgo-profiles" />
<project name="toolchain/polly" />
<project name="toolchain/ppl" />
<project name="toolchain/prebuilts/ndk-darwin/r21" />
<project name="toolchain/prebuilts/ndk-darwin/r23" />
<project name="toolchain/prebuilts/ndk/r13" />
<project name="toolchain/prebuilts/ndk/r14" />
<project name="toolchain/prebuilts/ndk/r15" />
<project name="toolchain/prebuilts/ndk/r16" />
<project name="toolchain/prebuilts/ndk/r17" />
<project name="toolchain/prebuilts/ndk/r18" />
<project name="toolchain/prebuilts/ndk/r19" />
<project name="toolchain/prebuilts/ndk/r20" />
<project name="toolchain/prebuilts/ndk/r21" />
<project name="toolchain/prebuilts/ndk/r23" />
<project name="toolchain/python" />
<project name="toolchain/rustc" />
<project name="toolchain/sed" />
<project name="toolchain/valgrind" />
<project name="toolchain/xz" />
<project name="toolchain/yasm" />
<project name="tools/aospstats" />
<project name="tools/manifest" />
<project name="tools/platform-compat" />
<project name="tools/presubmit-automerger/test1" />
<project name="tools/repo" />
<project name="trusty/app/avb" />
<project name="trusty/app/confirmationui" />
<project name="trusty/app/gatekeeper" />
<project name="trusty/app/keymaster" />
<project name="trusty/app/nvram" />
<project name="trusty/app/sample" />
<project name="trusty/app/storage" />
<project name="trusty/device/arm/generic-arm64" />
<project name="trusty/device/arm/vexpress-a15" />
<project name="trusty/device/nxp/imx6ul" />
<project name="trusty/device/nxp/imx7d" />
<project name="trusty/device/nxp/imx8m" />
<project name="trusty/device/x86/generic-x86_64" />
<project name="trusty/external/headers" />
<project name="trusty/external/musl" />
<project name="trusty/external/qemu" />
<project name="trusty/external/qemu-keycodemapdb" />
<project name="trusty/external/trusted-firmware-a" />
<project name="trusty/external/trusty" />
<project name="trusty/lib" />
<project name="trusty/lk/common" />
<project name="trusty/lk/nxp" />
<project name="trusty/lk/trusty" />
<project name="trusty/manifest" />
<project name="trusty/prebuilts/aosp" />
<project name="trusty/vendor/google/aosp" />

</manifest

