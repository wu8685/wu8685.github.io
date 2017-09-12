---
layout: post
title: Kubernetes Volume Manager
categories: [Kubernetes]
tags: [Kubernetes]
---

读了下kubelet下k8s volume manager的代码

![volume manager]({{ "/img/posts/2017-09-12-k8s-volume-manager.md/volume_manager.png" | prepend: site.baseurl }})

# 流程
1. volumeManager会启动两个goroutine：desiredStateOfWorldPopulator和reconciler。
2. 他们会去维护desired和actual两个volume-pod mapping信息，分别保存再desiredStateOfWorld和actualStateOfWorld。根据desire和actual的状态来做相应的动作：比如attach/detach volume和mount/unmount pod
3. 在最终调用docker创建容器的时候，会调用volumeManager的GetMountedVolumesForPod方法，从actualStateOfWorld里找到pod对应的mount信息，组成docker的mount参数传给docker daemon，创建容器

## desiredStateOfWorldPopulator
维护当前节点上所有pod所期望的volume mount状态。
由一个独立的goroutine for循环去定时更新状态，每100ms跑一次。
populator里主要通过添加和删除pod-volume mapping来维护desiredStateOfWorld的实例

### findAndAddNewPods()
输入：从statusManager里拿到当前的所有pod。
输出：用新的volume-pod mapping信息更新desiredStateOfWorld。

### findAndRemoveDeletedPods()
输入：遍历desiredStateOfWorld对象里的所有pod，通过podManager(pkg/kubelet/pod/pod_manager.go)和kubeContainerRuntime(pkg/kubelet/kuberuntime/kuberuntime_manager.go)拿到每个pod的最新状态
输出：当pod处于terminated状态，将pod的volume-pod mapping信息从desiredStateOfWorld里删除。

``` go
# 判断pod是terminated状态
func (dswp *desiredStateOfWorldPopulator) isPodTerminated(pod *v1.Pod) bool {
   podStatus, found := dswp.podStatusProvider.GetPodStatus(pod.UID)
   if !found {
      podStatus = pod.Status
   }
   return podStatus.Phase == v1.PodFailed || podStatus.Phase == v1.PodSucceeded || (pod.DeletionTimestamp != nil && notRunning(podStatus.ContainerStatuses))
}

// notRunning returns true if every status is terminated or waiting, or the status list
// is empty.
func notRunning(statuses []v1.ContainerStatus) bool {
   for _, status := range statuses {
      if status.State.Terminated == nil && status.State.Waiting == nil {
         return false
      }
   }
   return true
}
```

``` go
# desiredStateOfWorld 上维护了一个map，记录了当前期望的volume-pod mapping
type desiredStateOfWorld struct {
   // volumesToMount is a map containing the set of volumes that should be
   // attached to this node and mounted to the pods referencing it. The key in
   // the map is the name of the volume and the value is a volume object
   // containing more information about the volume.
   volumesToMount map[v1.UniqueVolumeName]volumeToMount
   // volumePluginMgr is the volume plugin manager used to create volume
   // plugin objects.
   volumePluginMgr *volume.VolumePluginMgr

   sync.RWMutex
}

// The volume object represents a volume that should be attached to this node,
// and mounted to podsToMount.
type volumeToMount struct {
   // volumeName contains the unique identifier for this volume.
   volumeName v1.UniqueVolumeName

   // podsToMount is a map containing the set of pods that reference this
   // volume and should mount it once it is attached. The key in the map is
   // the name of the pod and the value is a pod object containing more
   // information about the pod.
   podsToMount map[types.UniquePodName]podToMount

   // pluginIsAttachable indicates that the plugin for this volume implements
   // the volume.Attacher interface
   pluginIsAttachable bool

   // volumeGidValue contains the value of the GID annotation, if present.
   volumeGidValue string

   // reportedInUse indicates that the volume was successfully added to the
   // VolumesInUse field in the node's status.
   reportedInUse bool
}

// The pod object represents a pod that references the underlying volume and
// should mount it once it is attached.
type podToMount struct {
   // podName contains the name of this pod.
   podName types.UniquePodName

   // Pod to mount the volume to. Used to create NewMounter.
   pod *v1.Pod

   // volume spec containing the specification for this volume. Used to
   // generate the volume plugin object, and passed to plugin methods.
   // For non-PVC volumes this is the same as defined in the pod object. For
   // PVC volumes it is from the dereferenced PV object.
   spec *volume.Spec

   // outerVolumeSpecName is the volume.Spec.Name() of the volume as referenced
   // directly in the pod. If the volume was referenced through a persistent
   // volume claim, this contains the volume.Spec.Name() of the persistent
   // volume claim
   outerVolumeSpecName string
}
```

## reconciler
自己维护了一个actualStateOfWorld的实例，相对于desiredStateOfWorld，他代表了当前实际的volume mount状态。
其也是由一个独立的goroutine for循环去定时concile，每100ms跑一次。
输入：代表期望状态的desiredStateOfWorld对象和代表实际状态的actualStateOfWorld对象。
输出：通过比较actualStateOfWorld和desiredStateOfWorld去实际的调用volumePlugin的attach、mount、unmount操作。

reconciler里面主要运行reconcile和sync两个方法

### reconcile()
由三个for循环组成：

#### unmount循环
输入：actualStateOfWorld和desiredStateOfWorld的volume-pod信息
输出：对于在actualStateOfWorld而不在desiredStateOfWorld的volume-pod信息，执行unmount操作，这个操作结束后，**会将pod从对应的volume里删除**

#### attach循环
输入：actualStateOfWorld和desiredStateOfWorld的volume-pod信息
输出：对于同时存在于actualStateOfWorld和desiredStateOfWorld上的volume-pod信息：
* 如果volume没被attach过，则会去调用volume的attach方法，
* 否则，如果volume没被mount过，或者需要remount，则调用mount方法

#### unmountDevice循环
输入：存在于actualStateOfWorld中的，没有mount pod的volume
输出：如果此volume是globallyMounted的，则会调用deviceUnmount方法（此方法之后会标记volume的globallyMounted为false），否则调用detachVolume方法（此方法会尝试将没有对应pod的volume从actualStateOfWorld里删除）。globallyMounted是在reconciler的sync方法中，将所有attachable的volume都会标记为globallyMounted

#### sync()
通过读取主机上的目录/var/lib/kubelet/pods/volumes下的volume信息，然后将当前的volume信息更新到actualStateOfWorld

``` go
# actualStateOfWorld和desiredStateOfWorld类似，也是维护了一个volume-pod mapping信息
type actualStateOfWorld struct {
   // nodeName is the name of this node. This value is passed to Attach/Detach
   nodeName types.NodeName

   // attachedVolumes is a map containing the set of volumes the kubelet volume
   // manager believes to be successfully attached to this node. Volume types
   // that do not implement an attacher interface are assumed to be in this
   // state by default.
   // The key in this map is the name of the volume and the value is an object
   // containing more information about the attached volume.
   attachedVolumes map[v1.UniqueVolumeName]attachedVolume

   // volumePluginMgr is the volume plugin manager used to create volume
   // plugin objects.
   volumePluginMgr *volume.VolumePluginMgr
   sync.RWMutex
}

// attachedVolume represents a volume the kubelet volume manager believes to be
// successfully attached to a node it is managing. Volume types that do not
// implement an attacher are assumed to be in this state.
type attachedVolume struct {
   // volumeName contains the unique identifier for this volume.
   volumeName v1.UniqueVolumeName

   // mountedPods is a map containing the set of pods that this volume has been
   // successfully mounted to. The key in this map is the name of the pod and
   // the value is a mountedPod object containing more information about the
   // pod.
   mountedPods map[volumetypes.UniquePodName]mountedPod

   // spec is the volume spec containing the specification for this volume.
   // Used to generate the volume plugin object, and passed to plugin methods.
   // In particular, the Unmount method uses spec.Name() as the volumeSpecName
   // in the mount path:
   // /var/lib/kubelet/pods/{podUID}/volumes/{escapeQualifiedPluginName}/{volumeSpecName}/
   spec *volume.Spec

   // pluginName is the Unescaped Qualified name of the volume plugin used to
   // attach and mount this volume. It is stored separately in case the full
   // volume spec (everything except the name) can not be reconstructed for a
   // volume that should be unmounted (which would be the case for a mount path
   // read from disk without a full volume spec).
   pluginName string

   // pluginIsAttachable indicates the volume plugin used to attach and mount
   // this volume implements the volume.Attacher interface
   pluginIsAttachable bool

   // globallyMounted indicates that the volume is mounted to the underlying
   // device at a global mount point. This global mount point must be unmounted
   // prior to detach.
   globallyMounted bool

   // devicePath contains the path on the node where the volume is attached for
   // attachable volumes
   devicePath string
}

// The mountedPod object represents a pod for which the kubelet volume manager
// believes the underlying volume has been successfully been mounted.
type mountedPod struct {
   // the name of the pod
   podName volumetypes.UniquePodName

   // the UID of the pod
   podUID types.UID

   // mounter used to mount
   mounter volume.Mounter

   // outerVolumeSpecName is the volume.Spec.Name() of the volume as referenced
   // directly in the pod. If the volume was referenced through a persistent
   // volume claim, this contains the volume.Spec.Name() of the persistent
   // volume claim
   outerVolumeSpecName string

   // remountRequired indicates the underlying volume has been successfully
   // mounted to this pod but it should be remounted to reflect changes in the
   // referencing pod.
   // Atomically updating volumes depend on this to update the contents of the
   // volume. All volume mounting calls should be idempotent so a second mount
   // call for volumes that do not need to update contents should not fail.
   remountRequired bool

   // volumeGidValue contains the value of the GID annotation, if present.
   volumeGidValue string
}
```

## pendingOperations
因为volume的操作可能会有延迟，因此每个Volume操作都是用一个goroutine去执行，然后通过pendingOperations数据结构来cache，保证不会重复的执行同样的操作。
具体实现见：pkg/volume/util/nestedpendingoperations/nestedpendingoperations.go

# Volume Plugin
由volumeManager的图可以看到，最后都是去调用的volume plugin去具体的实现。每个内置的volume plugin的代码都是保存在pkg/volume下面，其中volume plugin的接口定义在pkg/volume/plugins.go

## 如何找到（注册）plugin
k8s内置的volume plugin在cmd/kubelet/app/plugin.go里面注册
自定义的plugin见文档：[official doc](https://github.com/kubernetes/community/blob/master/contributors/devel/flexvolume.md)

## git repo volume
配置：

``` go
volumes:
  - name: git-volume
    gitRepo:
      repository: "git@somewhere:me/my-git-repository.git"
      revision: "22f1d8406d464b0c0874075539c1f2e96c253775"	# Optional
      directory: "subdir"		# Optional: volume will mount path git@somewhere:me/my-git-repository.git/subdir
```

### 生命周期
因为git repo volume没有实现AttachableVolumePlugin接口，所以只有mount，unmount两个阶段。

#### mount
执行plugin的NewMounter.SetUp方法。
1. 执行git命令：
``` go
$ git clone <gitRepo.repository> [gitRepo.directory] /var/lib/kubelet/pods/<pod-uid>/volumes/kubernetes.io~git-repo/<volume-name>
```

2. 如果未设置gitRepo.revision，则结束。
3. 否则，继续执行：

``` go
$ git checkout <gitRepo.revision>
$ git reset --hard
```

``` go
// SetUp creates new directory and clones a git repo.
func (b *gitRepoVolumeMounter) SetUp(fsGroup *int64) error {
   return b.SetUpAt(b.GetPath(), fsGroup)
}

// SetUpAt creates new directory and clones a git repo.
func (b *gitRepoVolumeMounter) SetUpAt(dir string, fsGroup *int64) error {
   if volumeutil.IsReady(b.getMetaDir()) {
      return nil
   }

   // Wrap EmptyDir, let it do the setup.
   wrapped, err := b.plugin.host.NewWrapperMounter(b.volName, wrappedVolumeSpec(), &b.pod, b.opts)
   if err != nil {
      return err
   }
   if err := wrapped.SetUpAt(dir, fsGroup); err != nil {
      return err
   }

   args := []string{"clone", b.source}

   if len(b.target) != 0 {
      args = append(args, b.target)
   }
   if output, err := b.execCommand("git", args, dir); err != nil {
      return fmt.Errorf("failed to exec 'git %s': %s: %v",
         strings.Join(args, " "), output, err)
   }

   files, err := ioutil.ReadDir(dir)
   if err != nil {
      return err
   }

   if len(b.revision) == 0 {
      // Done!
      volumeutil.SetReady(b.getMetaDir())
      return nil
   }

   var subdir string

   switch {
   case b.target == ".":
      // if target dir is '.', use the current dir
      subdir = path.Join(dir)
   case len(files) == 1:
      // if target is not '.', use the generated folder
      subdir = path.Join(dir, files[0].Name())
   default:
      // if target is not '.', but generated many files, it's wrong
      return fmt.Errorf("unexpected directory contents: %v", files)
   }

   if output, err := b.execCommand("git", []string{"checkout", b.revision}, subdir); err != nil {
      return fmt.Errorf("failed to exec 'git checkout %s': %s: %v", b.revision, output, err)
   }
   if output, err := b.execCommand("git", []string{"reset", "--hard"}, subdir); err != nil {
      return fmt.Errorf("failed to exec 'git reset --hard': %s: %v", output, err)
   }

   volume.SetVolumeOwnership(b, fsGroup)

   volumeutil.SetReady(b.getMetaDir())
   return nil
}
```

#### unmount
1. volume名字接上".deleting~"
2. 清空该目录

# VolumeControllers on Kube-controller-manaer
* attachDetachController
* persistentVolumeController