---
title: Standard Input Output Over Socket
id: TRC-1
aliases: [
    "/tracker/trc1"
]
source: Tracker
icon: khulnasoft
shortName: Standard Input Output Over Socket
severity: high
draft: false
version: 0.1.0
keywords: "TRC-1"

category: runsec
date: 2021-04-15T20:55:39Z

remediations:

breadcrumbs: 
  - name: Tracker
    path: /tracker
  - name: Persistence
    path: /tracker/persistence

avd_page_type: avd_page
---

### Standard Input Output Over Socket
Redirection of process's standard input/output to socket

### MITRE ATT&CK
Persistence: Server Software Component


### Go Source
```
package main

import (
	"fmt"

	tracker "github.com/khulnasoft-lab/tracker/tracker-ebpf/external"
	"github.com/khulnasoft-lab/tracker/tracker-rules/types"
)

type stdioOverSocket struct {
	cb              types.SignatureHandler
	processSocketIp map[int]map[int]string
}

func (sig *stdioOverSocket) Init(cb types.SignatureHandler) error {
	sig.cb = cb
	sig.processSocketIp = make(map[int]map[int]string)

	return nil
}

func (sig *stdioOverSocket) GetMetadata() (types.SignatureMetadata, error) {
	return types.SignatureMetadata{
		ID:          "TRC-1",
		Version:     "0.1.0",
		Name:        "Standard Input/Output Over Socket",
		Description: "Redirection of process's standard input/output to socket",
		Tags:        []string{"linux", "container"},
		Properties: map[string]interface{}{
			"Severity":     3,
			"MITRE ATT&CK": "Persistence: Server Software Component",
		},
	}, nil
}

func (sig *stdioOverSocket) GetSelectedEvents() ([]types.SignatureEventSelector, error) {
	return []types.SignatureEventSelector{
		{Source: "tracker", Name: "connect"},
		{Source: "tracker", Name: "dup"},
		{Source: "tracker", Name: "dup2"},
		{Source: "tracker", Name: "dup3"},
		{Source: "tracker", Name: "close"},
		{Source: "tracker", Name: "sched_process_exit"},
	}, nil
}

func (sig *stdioOverSocket) OnEvent(e types.Event) error {

	eventObj, ok := e.(tracker.Event)
	if !ok {
		return fmt.Errorf("invalid event")
	}

	var connectData connectAddrData

	pid := eventObj.ProcessID

	switch eventObj.EventName {

	case "connect":

		sockfdArg, err := GetTrackerArgumentByName(eventObj, "sockfd")
		if err != nil {
			return err
		}

		sockfd := int(sockfdArg.Value.(int32))

		addrArg, err := GetTrackerArgumentByName(eventObj, "addr")
		if err != nil {
			return err
		}

		err = GetAddrStructFromArg(addrArg, &connectData)
		if err != nil {
			return err
		}

		if connectData.SaFamily == "AF_INET" {

			_, pidExists := sig.processSocketIp[pid]
			if !pidExists {
				sig.processSocketIp[pid] = make(map[int]string)
			}

			sig.processSocketIp[pid][sockfd] = connectData.SinAddr

		} else if connectData.SaFamily == "AF_INET6" {

			_, pidExists := sig.processSocketIp[pid]
			if !pidExists {
				sig.processSocketIp[pid] = make(map[int]string)
			}

			sig.processSocketIp[pid][sockfd] = connectData.SinAddr6
		}

	case "dup":

		pidSocketMap, pidExists := sig.processSocketIp[pid]

		if !pidExists {
			return nil
		}

		oldFdArg, err := GetTrackerArgumentByName(eventObj, "oldfd")
		if err != nil {
			return err
		}

		srcFd := int(oldFdArg.Value.(int32))

		dstFd := eventObj.ReturnValue

		err = isStdioOverSocket(sig, eventObj, pidSocketMap, srcFd, dstFd)
		if err != nil {
			return err
		}

	case "dup2", "dup3":

		pidSocketMap, pidExists := sig.processSocketIp[pid]

		if !pidExists {
			return nil
		}

		oldFdArg, err := GetTrackerArgumentByName(eventObj, "oldfd")
		if err != nil {
			return err
		}

		srcFd := int(oldFdArg.Value.(int32))

		newFdArg, err := GetTrackerArgumentByName(eventObj, "newfd")
		if err != nil {
			return err
		}

		dstFd := int(newFdArg.Value.(int32))

		err = isStdioOverSocket(sig, eventObj, pidSocketMap, srcFd, dstFd)
		if err != nil {
			return err
		}

	case "close":

		currentFdArg, err := GetTrackerArgumentByName(eventObj, "fd")
		if err != nil {
			return err
		}

		currentFd := int(currentFdArg.Value.(int32))

		delete(sig.processSocketIp[pid], currentFd)

	case "sched_process_exit":

		delete(sig.processSocketIp, pid)

	}

	return nil
}

func (sig *stdioOverSocket) OnSignal(s types.Signal) error {
	return nil
}

func isStdioOverSocket(sig *stdioOverSocket, eventObj tracker.Event, pidSocketMap map[int]string, srcFd int, dstFd int) error {

	stdAll := []int{0, 1, 2}

	ip, socketfdExists := pidSocketMap[srcFd]

	// this means that a socket FD is duplicated into one of the standard FDs
	if socketfdExists && intInSlice(dstFd, stdAll) {
		sig.cb(types.Finding{
			Context: eventObj,
			Data: map[string]interface{}{
				"ip": ip,
			},
		})
	}

	return nil
}

func intInSlice(a int, list []int) bool {
	for _, b := range list {
		if b == a {
			return true
		}
	}
	return false
}

```