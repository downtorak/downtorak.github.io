---
layout:     post
title:      WebRTC Receive Video Flow 분석
date:       2017-06-08 15:25:00 +0900
categories: WebRTC
summary:    따라가보자...
---

* content
{:toc}


### Before Start...
WebRTC Android M59 기준.

H264 처리를 기준.

Android MediaCodec Decoding은 Surface Texture를 사용.

Quick link 추가하기.




<br>

### BaseChannel.OnPacketRead()

**Function Information**

| File     | webrtc/pc/channel.cc |
| Class    | BaseChannel |
| Function | OnPacketRead() |
| Description | |

**Source Code**

```cpp
void BaseChannel::OnPacketRead(rtc::PacketTransportInternal* transport,
                               const char* data,
                               size_t len,
                               const rtc::PacketTime& packet_time,
                               int flags) {
  TRACE_EVENT0("webrtc", "BaseChannel::OnPacketRead");
  // OnPacketRead gets called from P2PSocket; now pass data to MediaEngine
  RTC_DCHECK(network_thread_->IsCurrent());

  // When using RTCP multiplexing we might get RTCP packets on the RTP
  // transport. We feed RTP traffic into the demuxer to determine if it is RTCP.
  bool rtcp = PacketIsRtcp(transport, data, len);
  rtc::CopyOnWriteBuffer packet(data, len);
  HandlePacket(rtcp, &packet, packet_time);  // (jominwoo): (Link [1])
}
```

**Call Function Links**

| [1] | [BaseChannel.HandlePacket()](#basechannelhandlepacket) |

<br>

### BaseChannel.HandlePacket()

**Function Information**

| File     | webrtc/pc/channel.cc |
| Class    | BaseChannel |
| Function | HandlePacket() |
| Description | |

**Source Code**

```cpp
void BaseChannel::HandlePacket(bool rtcp, rtc::CopyOnWriteBuffer* packet,
                               const rtc::PacketTime& packet_time) {
  RTC_DCHECK(network_thread_->IsCurrent());
  if (!WantsPacket(rtcp, packet)) {
    return;
  }

  // We are only interested in the first rtp packet because that
  // indicates the media has started flowing.
  if (!has_received_packet_ && !rtcp) {
    has_received_packet_ = true;
    signaling_thread()->Post(RTC_FROM_HERE, this, MSG_FIRSTPACKETRECEIVED);
  }

  // Unprotect the packet, if needed.
  if (srtp_filter_.IsActive()) {
    TRACE_EVENT0("webrtc", "SRTP Decode");
    char* data = packet->data<char>();
    int len = static_cast<int>(packet->size());
    bool res;
    if (!rtcp) {
      res = srtp_filter_.UnprotectRtp(data, len, &len);
      if (!res) {
        int seq_num = -1;
        uint32_t ssrc = 0;
        GetRtpSeqNum(data, len, &seq_num);
        GetRtpSsrc(data, len, &ssrc);
        LOG(LS_ERROR) << "Failed to unprotect " << content_name_
                      << " RTP packet: size=" << len
                      << ", seqnum=" << seq_num << ", SSRC=" << ssrc;
        return;
      }
    } else {
      res = srtp_filter_.UnprotectRtcp(data, len, &len);
      if (!res) {
        int type = -1;
        GetRtcpType(data, len, &type);
        LOG(LS_ERROR) << "Failed to unprotect " << content_name_
                      << " RTCP packet: size=" << len << ", type=" << type;
        return;
      }
    }

    packet->SetSize(len);
  } else if (srtp_required_) {
    // Our session description indicates that SRTP is required, but we got a
    // packet before our SRTP filter is active. This means either that
    // a) we got SRTP packets before we received the SDES keys, in which case
    //    we can't decrypt it anyway, or
    // b) we got SRTP packets before DTLS completed on both the RTP and RTCP
    //    transports, so we haven't yet extracted keys, even if DTLS did
    //    complete on the transport that the packets are being sent on. It's
    //    really good practice to wait for both RTP and RTCP to be good to go
    //    before sending  media, to prevent weird failure modes, so it's fine
    //    for us to just eat packets here. This is all sidestepped if RTCP mux
    //    is used anyway.
    LOG(LS_WARNING) << "Can't process incoming " << PacketType(rtcp)
                    << " packet when SRTP is inactive and crypto is required";
    return;
  }

  invoker_.AsyncInvoke<void>(
      RTC_FROM_HERE, worker_thread_,
      Bind(&BaseChannel::OnPacketReceived, this, rtcp, *packet, packet_time));  // (jominwoo): (Link [1])
}
```

**Call Function Links**

| [1] | [BaseChannel.OnPacketReceived()](#basechannelonpacketreceived) |

<br>

### BaseChannel.OnPacketReceived()

**Function Information**

| File     | webrtc/pc/channel.cc |
| Class    | BaseChannel |
| Function | OnPacketReceived() |
| Description | |

**Source Code**

```cpp
void BaseChannel::OnPacketReceived(bool rtcp,
                                   const rtc::CopyOnWriteBuffer& packet,
                                   const rtc::PacketTime& packet_time) {
  RTC_DCHECK(worker_thread_->IsCurrent());
  // Need to copy variable because OnRtcpReceived/OnPacketReceived
  // requires non-const pointer to buffer. This doesn't memcpy the actual data.
  rtc::CopyOnWriteBuffer data(packet);
  if (rtcp) {
    media_channel_->OnRtcpReceived(&data, packet_time);
  } else {
    media_channel_->OnPacketReceived(&data, packet_time);  // (jominwoo): (Link [1])
  }
}
```

**Call Function Links**

| [1] | [WebRtcVideoChannel2.OnPacketReceived()](#webrtcvideochannel2onpacketreceived) |

<br>

### WebRtcVideoChannel2.OnPacketReceived()

**Function Information**

| File     | webrtc/media/engine/webrtcvideoengine2.cc |
| Class    | WebRtcVideoChannel2 |
| Function | OnPacketReceived() |
| Description | |

**Source Code**

```cpp
void WebRtcVideoChannel2::OnPacketReceived(
    rtc::CopyOnWriteBuffer* packet,
    const rtc::PacketTime& packet_time) {
  const webrtc::PacketTime webrtc_packet_time(packet_time.timestamp,
                                              packet_time.not_before);
  const webrtc::PacketReceiver::DeliveryStatus delivery_result =
      call_->Receiver()->DeliverPacket(  // (jominwoo): (Link [1])
          webrtc::MediaType::VIDEO,
          packet->cdata(), packet->size(),
          webrtc_packet_time);
  switch (delivery_result) {
    case webrtc::PacketReceiver::DELIVERY_OK:
      return;
    case webrtc::PacketReceiver::DELIVERY_PACKET_ERROR:
      return;
    case webrtc::PacketReceiver::DELIVERY_UNKNOWN_SSRC:
      break;
  }

  uint32_t ssrc = 0;
  if (!GetRtpSsrc(packet->cdata(), packet->size(), &ssrc)) {
    return;
  }

  int payload_type = 0;
  if (!GetRtpPayloadType(packet->cdata(), packet->size(), &payload_type)) {
    return;
  }

  // See if this payload_type is registered as one that usually gets its own
  // SSRC (RTX) or at least is safe to drop either way (FEC). If it is, and
  // it wasn't handled above by DeliverPacket, that means we don't know what
  // stream it associates with, and we shouldn't ever create an implicit channel
  // for these.
  for (auto& codec : recv_codecs_) {
    if (payload_type == codec.rtx_payload_type ||
        payload_type == codec.ulpfec.red_rtx_payload_type ||
        payload_type == codec.ulpfec.ulpfec_payload_type ||
        payload_type == codec.flexfec_payload_type) {
      return;
    }
  }

  switch (unsignalled_ssrc_handler_->OnUnsignalledSsrc(this, ssrc)) {
    case UnsignalledSsrcHandler::kDropPacket:
      return;
    case UnsignalledSsrcHandler::kDeliverPacket:
      break;
  }

  if (call_->Receiver()->DeliverPacket(  // (jominwoo): (Link [1])
          webrtc::MediaType::VIDEO,
          packet->cdata(), packet->size(),
          webrtc_packet_time) != webrtc::PacketReceiver::DELIVERY_OK) {
    LOG(LS_WARNING) << "Failed to deliver RTP packet on re-delivery.";
    return;
  }
}
```

**Call Function Links**

| [1] | [Call.DeliverPacket()](#calldeliverpacket) |

<br>

### Call.DeliverPacket()

**Function Information**

| File     | webrtc/call/call.cc |
| Class    | Call |
| Function | DeliverPacket() |
| Description | |

**Source Code**

```cpp
PacketReceiver::DeliveryStatus Call::DeliverPacket(
    MediaType media_type,
    const uint8_t* packet,
    size_t length,
    const PacketTime& packet_time) {
  // TODO(solenberg): Tests call this function on a network thread, libjingle
  // calls on the worker thread. We should move towards always using a network
  // thread. Then this check can be enabled.
  // RTC_DCHECK(!configuration_thread_checker_.CalledOnValidThread());
  if (RtpHeaderParser::IsRtcp(packet, length))
    return DeliverRtcp(media_type, packet, length);

  return DeliverRtp(media_type, packet, length, packet_time);  // (jominwoo): (Link [1])
}
```

**Call Function Links**

| [1] | [Call.DeliverRtp()](#calldeliverrtp) |

<br>

### Call.DeliverRtp()

**Function Information**

| File     | webrtc/call/call.cc |
| Class    | Call |
| Function | DeliverRtp() |
| Description | |

**Source Code**

```cpp
PacketReceiver::DeliveryStatus Call::DeliverRtp(MediaType media_type,
                                                const uint8_t* packet,
                                                size_t length,
                                                const PacketTime& packet_time) {
  TRACE_EVENT0("webrtc", "Call::DeliverRtp");

  RTC_DCHECK(media_type == MediaType::AUDIO || media_type == MediaType::VIDEO);

  ReadLockScoped read_lock(*receive_crit_);
  // TODO(nisse): We should parse the RTP header only here, and pass
  // on parsed_packet to the receive streams.
  rtc::Optional<RtpPacketReceived> parsed_packet =
      ParseRtpPacket(packet, length, packet_time);

  if (!parsed_packet)
    return DELIVERY_PACKET_ERROR;

  NotifyBweOfReceivedPacket(*parsed_packet, media_type);

  uint32_t ssrc = parsed_packet->Ssrc();

  if (media_type == MediaType::AUDIO) {
    auto it = audio_receive_ssrcs_.find(ssrc);
    if (it != audio_receive_ssrcs_.end()) {
      received_bytes_per_second_counter_.Add(static_cast<int>(length));
      received_audio_bytes_per_second_counter_.Add(static_cast<int>(length));
      it->second->OnRtpPacket(*parsed_packet);
      event_log_->LogRtpHeader(kIncomingPacket, media_type, packet, length);
      return DELIVERY_OK;
    }
  }
  if (media_type == MediaType::VIDEO) {
    auto it = video_receive_ssrcs_.find(ssrc);
    if (it != video_receive_ssrcs_.end()) {
      received_bytes_per_second_counter_.Add(static_cast<int>(length));
      received_video_bytes_per_second_counter_.Add(static_cast<int>(length));
      it->second->OnRtpPacket(*parsed_packet);  // (jominwoo): (Link [1])

      // Deliver media packets to FlexFEC subsystem.
      auto it_bounds = flexfec_receive_ssrcs_media_.equal_range(ssrc);
      for (auto it = it_bounds.first; it != it_bounds.second; ++it)
        it->second->OnRtpPacket(*parsed_packet);

      event_log_->LogRtpHeader(kIncomingPacket, media_type, packet, length);
      return DELIVERY_OK;
    }
  }
  if (media_type == MediaType::VIDEO) {
    received_bytes_per_second_counter_.Add(static_cast<int>(length));
    // TODO(brandtr): Update here when FlexFEC supports protecting audio.
    received_video_bytes_per_second_counter_.Add(static_cast<int>(length));
    auto it = flexfec_receive_ssrcs_protection_.find(ssrc);
    if (it != flexfec_receive_ssrcs_protection_.end()) {
      it->second->OnRtpPacket(*parsed_packet);
      event_log_->LogRtpHeader(kIncomingPacket, media_type, packet, length);
      return DELIVERY_OK;
    }
  }
  return DELIVERY_UNKNOWN_SSRC;
}
```

**Call Function Links**

| [1] | [VideoReceiveStream.OnRtpPacket()](#videoreceivestreamonrtppacket) |

<br>

### VideoReceiveStream.OnRtpPacket()

**Function Information**

| File     | webrtc/video/video_receive_stream.cc |
| Class    | VideoReceiveStream |
| Function | OnRtpPacket() |
| Description | |

**Source Code**

```cpp
void VideoReceiveStream::OnRtpPacket(const RtpPacketReceived& packet) {
  rtp_stream_receiver_.OnRtpPacket(packet);  // (jominwoo): (Link [1])
}
```

**Call Function Links**

| [1] | [RtpStreamReceiver.OnRtpPacket()](#rtpstreamreceiveronrtppacket) |

<br>

### RtpStreamReceiver.OnRtpPacket()

**Function Information**

| File     | webrtc/video/rtp_stream_receiver.cc |
| Class    | RtpStreamReceiver |
| Function | OnRtpPacket() |
| Description | |

**Source Code**

```cpp
void RtpStreamReceiver::OnRtpPacket(const RtpPacketReceived& packet) {
  {
    rtc::CritScope lock(&receive_cs_);
    if (!receiving_) {
      return;
    }
  }

  int64_t now_ms = clock_->TimeInMilliseconds();

  {
    // Periodically log the RTP header of incoming packets.
    rtc::CritScope lock(&receive_cs_);
    if (now_ms - last_packet_log_ms_ > kPacketLogIntervalMs) {
      std::stringstream ss;
      ss << "Packet received on SSRC: " << packet.Ssrc()
         << " with payload type: " << static_cast<int>(packet.PayloadType())
         << ", timestamp: " << packet.Timestamp()
         << ", sequence number: " << packet.SequenceNumber()
         << ", arrival time: " << packet.arrival_time_ms();
      int32_t time_offset;
      if (packet.GetExtension<TransmissionOffset>(&time_offset)) {
        ss << ", toffset: " << time_offset;
      }
      uint32_t send_time;
      if (packet.GetExtension<AbsoluteSendTime>(&send_time)) {
        ss << ", abs send time: " << send_time;
      }
      LOG(LS_INFO) << ss.str();
      last_packet_log_ms_ = now_ms;
    }
  }

  // TODO(nisse): Delete use of GetHeader, but needs refactoring of
  // ReceivePacket and IncomingPacket methods below.
  RTPHeader header;
  packet.GetHeader(&header);

  header.payload_type_frequency = kVideoPayloadTypeFrequency;

  bool in_order = IsPacketInOrder(header);
  rtp_payload_registry_.SetIncomingPayloadType(header);
  ReceivePacket(packet.data(), packet.size(), header, in_order);  // (jominwoo): (Link [1])
  // Update receive statistics after ReceivePacket.
  // Receive statistics will be reset if the payload type changes (make sure
  // that the first packet is included in the stats).
  rtp_receive_statistics_->IncomingPacket(
      header, packet.size(), IsPacketRetransmitted(header, in_order));
}
```

**Call Function Links**

| [1] | [RtpStreamReceiver.ReceivePacket()](#rtpstreamreceiverreceivepacket) |

<br>

### RtpStreamReceiver.ReceivePacket()

**Function Information**

| File     | webrtc/video/rtp_stream_receiver.cc |
| Class    | RtpStreamReceiver |
| Function | ReceivePacket() |
| Description | |

**Source Code**

```cpp
bool RtpStreamReceiver::ReceivePacket(const uint8_t* packet,
                                      size_t packet_length,
                                      const RTPHeader& header,
                                      bool in_order) {
  if (rtp_payload_registry_.IsEncapsulated(header)) {
    return ParseAndHandleEncapsulatingHeader(packet, packet_length, header);
  }
  const uint8_t* payload = packet + header.headerLength;
  assert(packet_length >= header.headerLength);
  size_t payload_length = packet_length - header.headerLength;
  PayloadUnion payload_specific;
  if (!rtp_payload_registry_.GetPayloadSpecifics(header.payloadType,
                                                 &payload_specific)) {
    return false;
  }
  return rtp_receiver_->IncomingRtpPacket(header, payload, payload_length,  // (jominwoo): (Link [1])
                                          payload_specific, in_order);
}
```

**Call Function Links**

| [1] | [RtpReceiverImpl.IncomingRtpPacket()](#rtpreceiverimplincomingrtppacket) |

<br>

### RtpReceiverImpl.IncomingRtpPacket()

**Function Information**

| File     | webrtc/modules/rtp_rtcp/source/rtp_receiver_impl.cc |
| Class    | RtpReceiverImpl |
| Function | IncomingRtpPacket() |
| Description | |

**Source Code**

```cpp
bool RtpReceiverImpl::IncomingRtpPacket(
  const RTPHeader& rtp_header,
  const uint8_t* payload,
  size_t payload_length,
  PayloadUnion payload_specific,
  bool in_order) {
  // Trigger our callbacks.
  CheckSSRCChanged(rtp_header);

  int8_t first_payload_byte = payload_length > 0 ? payload[0] : 0;
  bool is_red = false;

  if (CheckPayloadChanged(rtp_header, first_payload_byte, &is_red,
                          &payload_specific) == -1) {
    if (payload_length == 0) {
      // OK, keep-alive packet.
      return true;
    }
    LOG(LS_WARNING) << "Receiving invalid payload type.";
    return false;
  }

  WebRtcRTPHeader webrtc_rtp_header;
  memset(&webrtc_rtp_header, 0, sizeof(webrtc_rtp_header));
  webrtc_rtp_header.header = rtp_header;
  CheckCSRC(webrtc_rtp_header);

  UpdateSources();

  size_t payload_data_length = payload_length - rtp_header.paddingLength;

  bool is_first_packet_in_frame = false;
  {
    rtc::CritScope lock(&critical_section_rtp_receiver_);
    if (HaveReceivedFrame()) {
      is_first_packet_in_frame =
          last_received_sequence_number_ + 1 == rtp_header.sequenceNumber &&
          last_received_timestamp_ != rtp_header.timestamp;
    } else {
      is_first_packet_in_frame = true;
    }
  }

  int32_t ret_val = rtp_media_receiver_->ParseRtpPacket(  // (jominwoo): (Link [1])
      &webrtc_rtp_header, payload_specific, is_red, payload, payload_length,
      clock_->TimeInMilliseconds(), is_first_packet_in_frame);

  if (ret_val < 0) {
    return false;
  }

  {
    rtc::CritScope lock(&critical_section_rtp_receiver_);

    last_receive_time_ = clock_->TimeInMilliseconds();
    last_received_payload_length_ = payload_data_length;

    if (in_order) {
      if (last_received_timestamp_ != rtp_header.timestamp) {
        last_received_timestamp_ = rtp_header.timestamp;
        last_received_frame_time_ms_ = clock_->TimeInMilliseconds();
      }
      last_received_sequence_number_ = rtp_header.sequenceNumber;
    }
  }
  return true;
}
```

**Call Function Links**

| [1] | [RTPReceiverVideo.ParseRtpPacket()](#rtpreceivervideoparsertppacket) |

<br>

### RTPReceiverVideo.ParseRtpPacket()

**Function Information**

| File     | webrtc/modules/rtp_rtcp/source/rtp_receiver_video.cc |
| Class    | RTPReceiverVideo |
| Function | ParseRtpPacket() |
| Description | |

**Source Code**

```cpp
int32_t RTPReceiverVideo::ParseRtpPacket(WebRtcRTPHeader* rtp_header,
                                         const PayloadUnion& specific_payload,
                                         bool is_red,
                                         const uint8_t* payload,
                                         size_t payload_length,
                                         int64_t timestamp_ms,
                                         bool is_first_packet) {
  TRACE_EVENT2(TRACE_DISABLED_BY_DEFAULT("webrtc_rtp"), "Video::ParseRtp",
               "seqnum", rtp_header->header.sequenceNumber, "timestamp",
               rtp_header->header.timestamp);
  rtp_header->type.Video.codec = specific_payload.Video.videoCodecType;

  RTC_DCHECK_GE(payload_length, rtp_header->header.paddingLength);
  const size_t payload_data_length =
      payload_length - rtp_header->header.paddingLength;

  if (payload == NULL || payload_data_length == 0) {
    return data_callback_->OnReceivedPayloadData(NULL, 0, rtp_header) == 0 ? 0
                                                                           : -1;
  }

  if (first_packet_received_()) {
    LOG(LS_INFO) << "Received first video RTP packet";
  }

  // We are not allowed to hold a critical section when calling below functions.
  std::unique_ptr<RtpDepacketizer> depacketizer(
      RtpDepacketizer::Create(rtp_header->type.Video.codec));
  if (depacketizer.get() == NULL) {
    LOG(LS_ERROR) << "Failed to create depacketizer.";
    return -1;
  }

  rtp_header->type.Video.is_first_packet_in_frame = is_first_packet;
  RtpDepacketizer::ParsedPayload parsed_payload;
  if (!depacketizer->Parse(&parsed_payload, payload, payload_data_length))
    return -1;

  rtp_header->frameType = parsed_payload.frame_type;
  rtp_header->type = parsed_payload.type;
  rtp_header->type.Video.rotation = kVideoRotation_0;

  // Retrieve the video rotation information.
  if (rtp_header->header.extension.hasVideoRotation) {
    rtp_header->type.Video.rotation =
        rtp_header->header.extension.videoRotation;
  }

  rtp_header->type.Video.playout_delay =
      rtp_header->header.extension.playout_delay;

  return data_callback_->OnReceivedPayloadData(parsed_payload.payload,  // (jominwoo): (Link [1])
                                               parsed_payload.payload_length,
                                               rtp_header) == 0
             ? 0
             : -1;
}
```

**Call Function Links**

| [1] | [RtpStreamReceiver.OnReceivedPayloadData()](#rtpstreamreceiveronreceivedpayloaddata) |

<br>

### RtpStreamReceiver.OnReceivedPayloadData()

**Function Information**

| File     | webrtc/video/rtp_stream_receiver.cc |
| Class    | RtpStreamReceiver |
| Function | OnReceivedPayloadData() |
| Description | |

**Source Code**

```cpp
int32_t RtpStreamReceiver::OnReceivedPayloadData(
    const uint8_t* payload_data,
    size_t payload_size,
    const WebRtcRTPHeader* rtp_header) {
  WebRtcRTPHeader rtp_header_with_ntp = *rtp_header;
  rtp_header_with_ntp.ntp_time_ms =
      ntp_estimator_.Estimate(rtp_header->header.timestamp);
  VCMPacket packet(payload_data, payload_size, rtp_header_with_ntp);
  packet.timesNacked =
      nack_module_ ? nack_module_->OnReceivedPacket(packet) : -1;

  // In the case of a video stream without picture ids and no rtx the
  // RtpFrameReferenceFinder will need to know about padding to
  // correctly calculate frame references.
  if (packet.sizeBytes == 0) {
    reference_finder_->PaddingReceived(packet.seqNum);
    return 0;
  }

  if (packet.codec == kVideoCodecH264) {
    // Only when we start to receive packets will we know what payload type
    // that will be used. When we know the payload type insert the correct
    // sps/pps into the tracker.
    if (packet.payloadType != last_payload_type_) {
      last_payload_type_ = packet.payloadType;
      InsertSpsPpsIntoTracker(packet.payloadType);
    }

    switch (tracker_.CopyAndFixBitstream(&packet)) {
      case video_coding::H264SpsPpsTracker::kRequestKeyframe:
        keyframe_request_sender_->RequestKeyFrame();
        FALLTHROUGH();
      case video_coding::H264SpsPpsTracker::kDrop:
        return 0;
      case video_coding::H264SpsPpsTracker::kInsert:
        break;
    }

  } else {
    uint8_t* data = new uint8_t[packet.sizeBytes];
    memcpy(data, packet.dataPtr, packet.sizeBytes);
    packet.dataPtr = data;
  }

  packet_buffer_->InsertPacket(&packet);  // (jominwoo): (Link [1])
  return 0;
}
```

**Call Function Links**

| [1] | [PacketBuffer.InsertPacket()](#packetbufferinsertpacket) |

<br>

### PacketBuffer.InsertPacket()

**Function Information**

| File     | webrtc/modules/video_coding/packet_buffer.cc |
| Class    | PacketBuffer |
| Function | InsertPacket() |
| Description | |

**Source Code**

```cpp
bool PacketBuffer::InsertPacket(VCMPacket* packet) {
  std::vector<std::unique_ptr<RtpFrameObject>> found_frames;
  {
    rtc::CritScope lock(&crit_);
    uint16_t seq_num = packet->seqNum;
    size_t index = seq_num % size_;

    if (!first_packet_received_) {
      first_seq_num_ = seq_num;
      first_packet_received_ = true;
    } else if (AheadOf(first_seq_num_, seq_num)) {
      // If we have explicitly cleared past this packet then it's old,
      // don't insert it.
      if (is_cleared_to_first_seq_num_) {
        delete[] packet->dataPtr;
        packet->dataPtr = nullptr;
        return false;
      }

      first_seq_num_ = seq_num;
    }

    if (sequence_buffer_[index].used) {
      // Duplicate packet, just delete the payload.
      if (data_buffer_[index].seqNum == packet->seqNum) {
        delete[] packet->dataPtr;
        packet->dataPtr = nullptr;
        return true;
      }

      // The packet buffer is full, try to expand the buffer.
      while (ExpandBufferSize() && sequence_buffer_[seq_num % size_].used) {
      }
      index = seq_num % size_;

      // Packet buffer is still full.
      if (sequence_buffer_[index].used) {
        delete[] packet->dataPtr;
        packet->dataPtr = nullptr;
        return false;
      }
    }

    sequence_buffer_[index].frame_begin = packet->is_first_packet_in_frame;
    sequence_buffer_[index].frame_end = packet->markerBit;
    sequence_buffer_[index].seq_num = packet->seqNum;
    sequence_buffer_[index].continuous = false;
    sequence_buffer_[index].frame_created = false;
    sequence_buffer_[index].used = true;
    data_buffer_[index] = *packet;
    packet->dataPtr = nullptr;

    found_frames = FindFrames(seq_num);
  }

  for (std::unique_ptr<RtpFrameObject>& frame : found_frames)
    received_frame_callback_->OnReceivedFrame(std::move(frame));  // (jominwoo): (Link [1])

  return true;
}
```

**Call Function Links**

| [1] | [RtpStreamReceiver.OnReceivedFrame()](#rtpstreamreceiveronreceivedframe) |

<br>

### RtpStreamReceiver.OnReceivedFrame()

**Function Information**

| File     | webrtc/video/rtp_stream_receiver.cc |
| Class    | RtpStreamReceiver |
| Function | OnReceivedFrame() |
| Description | |

**Source Code**

```cpp
void RtpStreamReceiver::OnReceivedFrame(
    std::unique_ptr<video_coding::RtpFrameObject> frame) {
  if (!frame->delayed_by_retransmission())
    timing_->IncomingTimestamp(frame->timestamp, clock_->TimeInMilliseconds());
  reference_finder_->ManageFrame(std::move(frame));  // (jominwoo): (Link [1])
}
```

**Call Function Links**

| [1] | [RtpFrameReferenceFinder.ManageFrame()](#rtpframereferencefindermanageframe) |

<br>

### RtpFrameReferenceFinder.ManageFrame()

**Function Information**

| File     | webrtc/modules/video_coding/rtp_frame_reference_finder.cc |
| Class    | RtpFrameReferenceFinder |
| Function | ManageFrame() |
| Description | |

**Source Code**

```cpp
void RtpFrameReferenceFinder::ManageFrame(
    std::unique_ptr<RtpFrameObject> frame) {
  rtc::CritScope lock(&crit_);

  // If we have cleared past this frame, drop it.
  if (cleared_to_seq_num_ != -1 &&
      AheadOf<uint16_t>(cleared_to_seq_num_, frame->first_seq_num())) {
    return;
  }

  switch (frame->codec_type()) {
    case kVideoCodecFlexfec:
    case kVideoCodecULPFEC:
    case kVideoCodecRED:
      RTC_NOTREACHED();
      break;
    case kVideoCodecVP8:
      ManageFrameVp8(std::move(frame));
      break;
    case kVideoCodecVP9:
      ManageFrameVp9(std::move(frame));
      break;
    // Since the EndToEndTests use kVicdeoCodecUnknow we treat it the same as
    // kVideoCodecGeneric.
    // TODO(philipel): Take a look at the EndToEndTests and see if maybe they
    //                 should be changed to use kVideoCodecGeneric instead.
    case kVideoCodecUnknown:
    case kVideoCodecH264:
    case kVideoCodecI420:
    case kVideoCodecGeneric:
      ManageFrameGeneric(std::move(frame), kNoPictureId);  // (jominwoo): (Link [1])
      break;
  }
}
```

**Call Function Links**

| [1] | [RtpFrameReferenceFinder.ManageFrameGeneric()](#rtpframereferencefindermanageframegeneric) |

<br>

### RtpFrameReferenceFinder.ManageFrameGeneric()

**Function Information**

| File     | webrtc/modules/video_coding/rtp_frame_reference_finder.cc |
| Class    | RtpFrameReferenceFinder |
| Function | ManageFrameGeneric() |
| Description | |

**Source Code**

```cpp
void RtpFrameReferenceFinder::ManageFrameGeneric(
    std::unique_ptr<RtpFrameObject> frame,
    int picture_id) {
  // If |picture_id| is specified then we use that to set the frame references,
  // otherwise we use sequence number.
  if (picture_id != kNoPictureId) {
    if (last_unwrap_ == -1)
      last_unwrap_ = picture_id;

    frame->picture_id = UnwrapPictureId(picture_id % kPicIdLength);
    frame->num_references = frame->frame_type() == kVideoFrameKey ? 0 : 1;
    frame->references[0] = frame->picture_id - 1;
    frame_callback_->OnCompleteFrame(std::move(frame));  // (jominwoo): (Link [1])
    return;
  }

  if (frame->frame_type() == kVideoFrameKey) {
    last_seq_num_gop_.insert(std::make_pair(
        frame->last_seq_num(),
        std::make_pair(frame->last_seq_num(), frame->last_seq_num())));
  }

  // We have received a frame but not yet a keyframe, stash this frame.
  if (last_seq_num_gop_.empty()) {
    stashed_frames_.push_back(std::move(frame));
    return;
  }

  // Clean up info for old keyframes but make sure to keep info
  // for the last keyframe.
  auto clean_to = last_seq_num_gop_.lower_bound(frame->last_seq_num() - 100);
  for (auto it = last_seq_num_gop_.begin();
       it != clean_to && last_seq_num_gop_.size() > 1;) {
    it = last_seq_num_gop_.erase(it);
  }

  // Find the last sequence number of the last frame for the keyframe
  // that this frame indirectly references.
  auto seq_num_it = last_seq_num_gop_.upper_bound(frame->last_seq_num());
  if (seq_num_it == last_seq_num_gop_.begin()) {
    LOG(LS_WARNING) << "Generic frame with packet range ["
                    << frame->first_seq_num() << ", " << frame->last_seq_num()
                    << "] has no GoP, dropping frame.";
    return;
  }
  seq_num_it--;

  // Make sure the packet sequence numbers are continuous, otherwise stash
  // this frame.
  uint16_t last_picture_id_gop = seq_num_it->second.first;
  uint16_t last_picture_id_with_padding_gop = seq_num_it->second.second;
  if (frame->frame_type() == kVideoFrameDelta) {
    uint16_t prev_seq_num = frame->first_seq_num() - 1;
    if (prev_seq_num != last_picture_id_with_padding_gop) {
      stashed_frames_.push_back(std::move(frame));
      return;
    }
  }

  RTC_DCHECK(AheadOrAt(frame->last_seq_num(), seq_num_it->first));

  // Since keyframes can cause reordering we can't simply assign the
  // picture id according to some incrementing counter.
  frame->picture_id = frame->last_seq_num();
  frame->num_references = frame->frame_type() == kVideoFrameDelta;
  frame->references[0] = last_picture_id_gop;
  if (AheadOf(frame->picture_id, last_picture_id_gop)) {
    seq_num_it->second.first = frame->picture_id;
    seq_num_it->second.second = frame->picture_id;
  }

  last_picture_id_ = frame->picture_id;
  UpdateLastPictureIdWithPadding(frame->picture_id);
  frame_callback_->OnCompleteFrame(std::move(frame));
  RetryStashedFrames();
}
```

**Call Function Links**

| [1] | [RtpStreamReceiver.OnCompleteFrame()](#rtpstreamreceiveroncompleteframe) |

<br>

### RtpStreamReceiver.OnCompleteFrame()

**Function Information**

| File     | webrtc/video/rtp_stream_receiver.cc |
| Class    | RtpStreamReceiver |
| Function | OnCompleteFrame() |
| Description | |

**Source Code**

```cpp
void RtpStreamReceiver::OnCompleteFrame(
    std::unique_ptr<video_coding::FrameObject> frame) {
  {
    rtc::CritScope lock(&last_seq_num_cs_);
    video_coding::RtpFrameObject* rtp_frame =
        static_cast<video_coding::RtpFrameObject*>(frame.get());
    last_seq_num_for_pic_id_[rtp_frame->picture_id] = rtp_frame->last_seq_num();
  }
  complete_frame_callback_->OnCompleteFrame(std::move(frame));  // (jominwoo): (Link [1])
}
```

**Call Function Links**

| [1] | [VideoReceiveStream.OnCompleteFrame()](#videoreceivestreamoncompleteframe) |
| [2] | [Class.Function()](#classfunction) |
| [3] | [Class.Function()](#classfunction) |
| [4] | [Class.Function()](#classfunction) |

<br>

### VideoReceiveStream.OnCompleteFrame()

**Function Information**

| File     | webrtc/video/video_receive_stream.cc |
| Class    | VideoReceiveStream |
| Function | OnCompleteFrame() |
| Description | |

**Source Code**

```cpp
void VideoReceiveStream::OnCompleteFrame(
    std::unique_ptr<video_coding::FrameObject> frame) {
  int last_continuous_pid = frame_buffer_->InsertFrame(std::move(frame));  // (jominwoo): (Link [1])
  if (last_continuous_pid != -1)
    rtp_stream_receiver_.FrameContinuous(last_continuous_pid);
}
```

**Call Function Links**

| [1] | [FrameBuffer.InsertFrame()](#framebufferinsertframe) |

<br>

### FrameBuffer.InsertFrame()

**Function Information**

| File     | webrtc/modules/video_coding/frame_buffer2.cc |
| Class    | FrameBuffer |
| Function | InsertFrame() |
| Description | |

**Source Code**

```cpp
int FrameBuffer::InsertFrame(std::unique_ptr<FrameObject> frame) {
  TRACE_EVENT0("webrtc", "FrameBuffer::InsertFrame");
  RTC_DCHECK(frame);
  if (stats_callback_)
    stats_callback_->OnCompleteFrame(frame->num_references == 0, frame->size());
  FrameKey key(frame->picture_id, frame->spatial_layer);

  rtc::CritScope lock(&crit_);

  int last_continuous_picture_id =
      last_continuous_frame_it_ == frames_.end()
          ? -1
          : last_continuous_frame_it_->first.picture_id;

  if (num_frames_buffered_ >= kMaxFramesBuffered) {
    LOG(LS_WARNING) << "Frame with (picture_id:spatial_id) (" << key.picture_id
                    << ":" << static_cast<int>(key.spatial_layer)
                    << ") could not be inserted due to the frame "
                    << "buffer being full, dropping frame.";
    return last_continuous_picture_id;
  }

  if (frame->inter_layer_predicted && frame->spatial_layer == 0) {
    LOG(LS_WARNING) << "Frame with (picture_id:spatial_id) (" << key.picture_id
                    << ":" << static_cast<int>(key.spatial_layer)
                    << ") is marked as inter layer predicted, dropping frame.";
    return last_continuous_picture_id;
  }

  if (last_decoded_frame_it_ != frames_.end() &&
      key < last_decoded_frame_it_->first) {
    if (AheadOf(frame->timestamp, last_decoded_frame_timestamp_) &&
        frame->num_references == 0) {
      // If this frame has a newer timestamp but an earlier picture id then we
      // assume there has been a jump in the picture id due to some encoder
      // reconfiguration or some other reason. Even though this is not according
      // to spec we can still continue to decode from this frame if it is a
      // keyframe.
      LOG(LS_WARNING) << "A jump in picture id was detected, clearing buffer.";
      ClearFramesAndHistory();
      last_continuous_picture_id = -1;
    } else {
      LOG(LS_WARNING) << "Frame with (picture_id:spatial_id) ("
                      << key.picture_id << ":"
                      << static_cast<int>(key.spatial_layer)
                      << ") inserted after frame ("
                      << last_decoded_frame_it_->first.picture_id << ":"
                      << static_cast<int>(
                             last_decoded_frame_it_->first.spatial_layer)
                      << ") was handed off for decoding, dropping frame.";
      return last_continuous_picture_id;
    }
  }

  // Test if inserting this frame would cause the order of the frames to become
  // ambiguous (covering more than half the interval of 2^16). This can happen
  // when the picture id make large jumps mid stream.
  if (!frames_.empty() &&
      key < frames_.begin()->first &&
      frames_.rbegin()->first < key) {
    LOG(LS_WARNING) << "A jump in picture id was detected, clearing buffer.";
    ClearFramesAndHistory();
    last_continuous_picture_id = -1;
  }

  auto info = frames_.insert(std::make_pair(key, FrameInfo())).first;

  if (info->second.frame) {
    LOG(LS_WARNING) << "Frame with (picture_id:spatial_id) (" << key.picture_id
                    << ":" << static_cast<int>(key.spatial_layer)
                    << ") already inserted, dropping frame.";
    return last_continuous_picture_id;
  }

  if (!UpdateFrameInfoWithIncomingFrame(*frame, info))
    return last_continuous_picture_id;
  UpdatePlayoutDelays(*frame);
  info->second.frame = std::move(frame);
  // (jominwoo): The frame is pushed to FrameBuffer. Let's go to the point pulling the frame. (Link [1])
  ++num_frames_buffered_;

  if (info->second.num_missing_continuous == 0) {
    info->second.continuous = true;
    PropagateContinuity(info);
    last_continuous_picture_id = last_continuous_frame_it_->first.picture_id;

    // Since we now have new continuous frames there might be a better frame
    // to return from NextFrame. Signal that thread so that it again can choose
    // which frame to return.
    new_continuous_frame_event_.Set();
  }

  return last_continuous_picture_id;
}
```

**Call Function Links**

| [1] | [VideoReceiveStream.Decode()](#videoreceivestreamdecode) |

<br>

### VideoReceiveStream.Decode()

**Function Information**

| File     | webrtc/video/video_receive_stream.cc |
| Class    | VideoReceiveStream |
| Function | Decode() |
| Description | |

**Source Code**

```cpp
bool VideoReceiveStream::Decode() {
  TRACE_EVENT0("webrtc", "VideoReceiveStream::Decode");
  static const int kMaxWaitForFrameMs = 3000;
  std::unique_ptr<video_coding::FrameObject> frame;
  video_coding::FrameBuffer::ReturnReason res =
      frame_buffer_->NextFrame(kMaxWaitForFrameMs, &frame);

  if (res == video_coding::FrameBuffer::ReturnReason::kStopped) {
    video_receiver_.DecodingStopped();
    return false;
  }

  if (frame) {
    if (video_receiver_.Decode(frame.get()) == VCM_OK)  // (jominwoo): (Link [1])
      rtp_stream_receiver_.FrameDecoded(frame->picture_id);
  } else {
    LOG(LS_WARNING) << "No decodable frame in " << kMaxWaitForFrameMs
                    << " ms, requesting keyframe.";
    RequestKeyFrame();
  }
  return true;
}
```

**Call Function Links**

| [1] | [VideoReceiver.Decode()](#videoreceiverdecode-1) |
| [2] | [Class.Function()](#classfunction) |
| [3] | [Class.Function()](#classfunction) |
| [4] | [Class.Function()](#classfunction) |

<br>

### VideoReceiver.Decode()-1

**Function Information**

| File     | webrtc/modules/video_coding/video_receiver.cc |
| Class    | VideoReceiver |
| Function | Decode() |
| Description | |

**Source Code**

```cpp
int32_t VideoReceiver::Decode(const webrtc::VCMEncodedFrame* frame) {
  rtc::CritScope lock(&receive_crit_);
  if (pre_decode_image_callback_) {
    EncodedImage encoded_image(frame->EncodedImage());
    int qp = -1;
    if (qp_parser_.GetQp(*frame, &qp)) {
      encoded_image.qp_ = qp;
    }
    pre_decode_image_callback_->OnEncodedImage(encoded_image,
                                               frame->CodecSpecific(), nullptr);
  }
  return Decode(*frame);  // (jominwoo): (Link [1])
}
```

**Call Function Links**

| [1] | [VideoReceiver.Decode()](#videoreceiverdecode-2) |

<br>

### VideoReceiver.Decode()-2

**Function Information**

| File     | webrtc/modules/video_coding/video_receiver.cc |
| Class    | VideoReceiver |
| Function | Decode() |
| Description | |

**Source Code**

```cpp
int32_t VideoReceiver::Decode(const VCMEncodedFrame& frame) {
  TRACE_EVENT0("webrtc", "VideoReceiver::Decode");
  // Change decoder if payload type has changed
  VCMGenericDecoder* decoder =
      _codecDataBase.GetDecoder(frame, &_decodedFrameCallback);
  if (decoder == nullptr) {
    return VCM_NO_CODEC_REGISTERED;
  }
  // Decode a frame
  int32_t ret = decoder->Decode(frame, clock_->TimeInMilliseconds());  // (jominwoo): (Link [1])

  // Check for failed decoding, run frame type request callback if needed.
  bool request_key_frame = false;
  if (ret < 0) {
    request_key_frame = true;
  }

  if (!frame.Complete() || frame.MissingFrame()) {
    request_key_frame = true;
    ret = VCM_OK;
  }
  if (request_key_frame) {
    rtc::CritScope cs(&process_crit_);
    _scheduleKeyRequest = true;
  }
  return ret;
}
```

**Call Function Links**

| [1] | [VCMGenericDecoder.Decode()](#vcmgenericdecoderdecode) |

<br>

### VCMGenericDecoder.Decode()

**Function Information**

| File     | webrtc/modules/video_coding/generic_decoder.cc |
| Class    | VCMGenericDecoder |
| Function | Decode() |
| Description | |

**Source Code**

```cpp
int32_t VCMGenericDecoder::Decode(const VCMEncodedFrame& frame, int64_t nowMs) {
    TRACE_EVENT1("webrtc", "VCMGenericDecoder::Decode", "timestamp",
                 frame.EncodedImage()._timeStamp);
    _frameInfos[_nextFrameInfoIdx].decodeStartTimeMs = nowMs;
    _frameInfos[_nextFrameInfoIdx].renderTimeMs = frame.RenderTimeMs();
    _frameInfos[_nextFrameInfoIdx].rotation = frame.rotation();
    _callback->Map(frame.TimeStamp(), &_frameInfos[_nextFrameInfoIdx]);

    _nextFrameInfoIdx = (_nextFrameInfoIdx + 1) % kDecoderFrameMemoryLength;
    const RTPFragmentationHeader dummy_header;
    int32_t ret = _decoder->Decode(frame.EncodedImage(), frame.MissingFrame(),  // (jominwoo): (Link [1])
                                   &dummy_header,
                                   frame.CodecSpecific(), frame.RenderTimeMs());

    _callback->OnDecoderImplementationName(_decoder->ImplementationName());
    if (ret < WEBRTC_VIDEO_CODEC_OK) {
        LOG(LS_WARNING) << "Failed to decode frame with timestamp "
                        << frame.TimeStamp() << ", error code: " << ret;
        _callback->Pop(frame.TimeStamp());
        return ret;
    } else if (ret == WEBRTC_VIDEO_CODEC_NO_OUTPUT ||
               ret == WEBRTC_VIDEO_CODEC_REQUEST_SLI) {
        // No output
        _callback->Pop(frame.TimeStamp());
    }
    return ret;
}
```

**Call Function Links**

| [1] | [MediaCodecVideoDecoder.Decode()](#mediacodecvideodecoderdecode) |
| [2] | [Class.Function()](#classfunction) |
| [3] | [Class.Function()](#classfunction) |
| [4] | [Class.Function()](#classfunction) |

<br>

### MediaCodecVideoDecoder.Decode()

**Function Information**

| File     | webrtc/sdk/android/src/jni/androidmediadecoder_jni.cc |
| Class    | MediaCodecVideoDecoder |
| Function | Decode() |
| Description | |

**Source Code**

```cpp
int32_t MediaCodecVideoDecoder::Decode(
    const EncodedImage& inputImage,
    bool missingFrames,
    const RTPFragmentationHeader* fragmentation,
    const CodecSpecificInfo* codecSpecificInfo,
    int64_t renderTimeMs) {
  // (jominwoo): 예외처리
  if (sw_fallback_required_) {
    ALOGE << "Decode() - fallback to SW codec";
    return WEBRTC_VIDEO_CODEC_FALLBACK_SOFTWARE;
  }
  if (callback_ == NULL) {
    ALOGE << "Decode() - callback_ is NULL";
    return WEBRTC_VIDEO_CODEC_UNINITIALIZED;
  }
  if (inputImage._buffer == NULL && inputImage._length > 0) {
    ALOGE << "Decode() - inputImage is incorrect";
    return WEBRTC_VIDEO_CODEC_ERR_PARAMETER;
  }
  if (!inited_) {
    ALOGE << "Decode() - decoder is not initialized";
    return WEBRTC_VIDEO_CODEC_UNINITIALIZED;
  }

  // (jominwoo): Resolution이 바뀌면 Decoder Reset 수행
  // Check if encoded frame dimension has changed.
  if ((inputImage._encodedWidth * inputImage._encodedHeight > 0) &&
      (inputImage._encodedWidth != codec_.width ||
      inputImage._encodedHeight != codec_.height)) {
    ALOGW << "Input resolution changed from " <<
        codec_.width << " x " << codec_.height << " to " <<
        inputImage._encodedWidth << " x " << inputImage._encodedHeight;
    codec_.width = inputImage._encodedWidth;
    codec_.height = inputImage._encodedHeight;
    int32_t ret;
    if (use_surface_ &&
        (codecType_ == kVideoCodecVP8 || codecType_ == kVideoCodecH264)) {
      // Soft codec reset - only for surface decoding.
      ret = codec_thread_->Invoke<int32_t>(
          RTC_FROM_HERE,
          Bind(&MediaCodecVideoDecoder::ResetDecodeOnCodecThread, this));
    } else {
      // Hard codec reset.
      ret = InitDecode(&codec_, 1);
    }
    if (ret < 0) {
      ALOGE << "InitDecode failure: " << ret << " - fallback to SW codec";
      sw_fallback_required_ = true;
      return WEBRTC_VIDEO_CODEC_FALLBACK_SOFTWARE;
    }
  }

  // (jominwoo): Key Frame 필요할 경우 Key Frame이 올때까지 버림
  // Always start with a complete key frame.
  if (key_frame_required_) {
    if (inputImage._frameType != webrtc::kVideoFrameKey) {
      ALOGE << "Decode() - key frame is required";
      return WEBRTC_VIDEO_CODEC_ERROR;
    }
    if (!inputImage._completeFrame) {
      ALOGE << "Decode() - complete frame is required";
      return WEBRTC_VIDEO_CODEC_ERROR;
    }
    key_frame_required_ = false;
  }
  // (jominwoo): 데이터가 없는 Frame에 대한 예외처리?
  if (inputImage._length == 0) {
    return WEBRTC_VIDEO_CODEC_ERROR;
  }

  // (jominwoo): codec_thread에서 DecodeOnCodecThread 함수 실행 (Link [1])
  return codec_thread_->Invoke<int32_t>(
      RTC_FROM_HERE,
      Bind(&MediaCodecVideoDecoder::DecodeOnCodecThread, this, inputImage));
}
```

**Call Function Links**

| [1] | [MediaCodecVideoDecoder.DecodeOnCodecThread()](#mediacodecvideodecoderdecodeoncodecthread) |

<br>

### MediaCodecVideoDecoder.DecodeOnCodecThread()

**Function Information**

| File     | webrtc/sdk/android/src/jni/androidmediadecoder_jni.cc |
| Class    | MediaCodecVideoDecoder |
| Function | DecodeOnCodecThread() |
| Description | |

**Source Code**

```cpp
int32_t MediaCodecVideoDecoder::DecodeOnCodecThread(
    const EncodedImage& inputImage) {
  CheckOnCodecThread();
  JNIEnv* jni = AttachCurrentThreadIfNeeded();
  ScopedLocalRefFrame local_ref_frame(jni);

  // Try to drain the decoder and wait until output is not too
  // much behind the input.
  if (codecType_ == kVideoCodecH264 &&
      frames_received_ > frames_decoded_ + max_pending_frames_) {
    // Print warning for H.264 only - for VP8/VP9 one frame delay is ok.
    ALOGW << "Decoder is too far behind. Try to drain. Received: " <<
        frames_received_ << ". Decoded: " << frames_decoded_;
    EnableFrameLogOnWarning();
  }
  const int64 drain_start = rtc::TimeMillis();
  // (jominwoo): 받은 Frame수와 디코딩한 Frame수 차이가
  //             max_pending_frames_ 보다 크고
  //             drain을 시작한지 kMediaCodecTimeoutMs만큼이 안되었으면 drain
  while ((frames_received_ > frames_decoded_ + max_pending_frames_) &&
         (rtc::TimeMillis() - drain_start) < kMediaCodecTimeoutMs) {
    if (!DeliverPendingOutputs(jni, kMediaCodecPollMs)) {
      ALOGE << "DeliverPendingOutputs error. Frames received: " <<
          frames_received_ << ". Frames decoded: " << frames_decoded_;
      return ProcessHWErrorOnCodecThread();
    }
  }
  if (frames_received_ > frames_decoded_ + max_pending_frames_) {
    ALOGE << "Output buffer dequeue timeout. Frames received: " <<
        frames_received_ << ". Frames decoded: " << frames_decoded_;
    return ProcessHWErrorOnCodecThread();
  }

  // Get input buffer.
  int j_input_buffer_index = jni->CallIntMethod(
      *j_media_codec_video_decoder_, j_dequeue_input_buffer_method_);
  if (CheckException(jni) || j_input_buffer_index < 0) {
    ALOGE << "dequeueInputBuffer error: " << j_input_buffer_index <<
        ". Retry DeliverPendingOutputs.";
    EnableFrameLogOnWarning();
    // Try to drain the decoder.
    if (!DeliverPendingOutputs(jni, kMediaCodecPollMs)) {
      ALOGE << "DeliverPendingOutputs error. Frames received: " <<
          frames_received_ << ". Frames decoded: " << frames_decoded_;
      return ProcessHWErrorOnCodecThread();
    }
    // Try dequeue input buffer one last time.
    j_input_buffer_index = jni->CallIntMethod(
        *j_media_codec_video_decoder_, j_dequeue_input_buffer_method_);
    if (CheckException(jni) || j_input_buffer_index < 0) {
      ALOGE << "dequeueInputBuffer critical error: " << j_input_buffer_index;
      return ProcessHWErrorOnCodecThread();
    }
  }

  // Copy encoded data to Java ByteBuffer.
  jobject j_input_buffer = input_buffers_[j_input_buffer_index];
  uint8_t* buffer =
      reinterpret_cast<uint8_t*>(jni->GetDirectBufferAddress(j_input_buffer));
  RTC_CHECK(buffer) << "Indirect buffer??";
  int64_t buffer_capacity = jni->GetDirectBufferCapacity(j_input_buffer);
  if (CheckException(jni) || buffer_capacity < inputImage._length) {
    ALOGE << "Input frame size "<<  inputImage._length <<
        " is bigger than buffer size " << buffer_capacity;
    return ProcessHWErrorOnCodecThread();
  }
  jlong presentation_timestamp_us = static_cast<jlong>(
      static_cast<int64_t>(frames_received_) * 1000000 / codec_.maxFramerate);
  memcpy(buffer, inputImage._buffer, inputImage._length);

  if (frames_decoded_ < frames_decoded_logged_) {
    ALOGD << "Decoder frame in # " << frames_received_ <<
        ". Type: " << inputImage._frameType <<
        ". Buffer # " << j_input_buffer_index <<
        ". TS: " << presentation_timestamp_us / 1000 <<
        ". Size: " << inputImage._length;
  }

  // Save input image timestamps for later output.
  frames_received_++;
  current_bytes_ += inputImage._length;
  rtc::Optional<uint8_t> qp;
  if (codecType_ == kVideoCodecVP8) {
    int qp_int;
    if (webrtc::vp8::GetQp(inputImage._buffer, inputImage._length, &qp_int)) {
      qp = rtc::Optional<uint8_t>(qp_int);
    }
  } else if (codecType_ == kVideoCodecH264) {
    h264_bitstream_parser_.ParseBitstream(inputImage._buffer,
                                          inputImage._length);
    int qp_int;
    if (h264_bitstream_parser_.GetLastSliceQp(&qp_int)) {
      qp = rtc::Optional<uint8_t>(qp_int);
    }
  }
  pending_frame_qps_.push_back(qp);

  // Feed input to decoder.
  bool success = jni->CallBooleanMethod(
      *j_media_codec_video_decoder_,
      j_queue_input_buffer_method_,  // (jominwoo): (Link [1])
      j_input_buffer_index,
      inputImage._length,
      presentation_timestamp_us,
      static_cast<int64_t> (inputImage._timeStamp),
      inputImage.ntp_time_ms_);
  if (CheckException(jni) || !success) {
    ALOGE << "queueInputBuffer error";
    return ProcessHWErrorOnCodecThread();
  }

  // Try to drain the decoder
  if (!DeliverPendingOutputs(jni, 0)) {  // (jominwoo): (Link [2])
    ALOGE << "DeliverPendingOutputs error";
    return ProcessHWErrorOnCodecThread();
  }

  return WEBRTC_VIDEO_CODEC_OK;
}
```

**Call Function Links**

| [1] | [MediaCodecVideoDecoder.queueInputBuffer()](#mediacodecvideodecoderqueueinputbuffer) |
| [2] | [MediaCodecVideoDecoder.DeliverPendingOutputs()](#mediacodecvideodecoderdeliverpendingoutputs) |

<br>

### MediaCodecVideoDecoder.queueInputBuffer()

**Function Information**

| File     | webrtc/sdk/android/api/org/webrtc/MediaCodecVideoDecoder.java |
| Class    | MediaCodecVideoDecoder |
| Function | queueInputBuffer() |
| Description | |

**Source Code**

```java
  private boolean queueInputBuffer(int inputBufferIndex, int size, long presentationTimeStamUs,
      long timeStampMs, long ntpTimeStamp) {
    checkOnMediaCodecThread();
    try {
      inputBuffers[inputBufferIndex].position(0);
      inputBuffers[inputBufferIndex].limit(size);
      decodeStartTimeMs.add(
          new TimeStamps(SystemClock.elapsedRealtime(), timeStampMs, ntpTimeStamp));
      mediaCodec.queueInputBuffer(inputBufferIndex, 0, size, presentationTimeStamUs, 0);
      return true;
    } catch (IllegalStateException e) {
      Logging.e(TAG, "decode failed", e);
      return false;
    }
  }
```

<br>

### MediaCodecVideoDecoder.DeliverPendingOutputs()

**Function Information**

| File     | webrtc/sdk/android/src/jni/androidmediadecoder_jni.cc |
| Class    | MediaCodecVideoDecoder |
| Function | DeliverPendingOutputs() |
| Description | |

**Source Code**

```cpp
bool MediaCodecVideoDecoder::DeliverPendingOutputs(
    JNIEnv* jni, int dequeue_timeout_ms) {
  CheckOnCodecThread();
  if (frames_received_ <= frames_decoded_) {
    // No need to query for output buffers - decoder is drained.
    return true;
  }
  // Get decoder output.
  jobject j_decoder_output_buffer =
      jni->CallObjectMethod(*j_media_codec_video_decoder_,
          use_surface_ ? j_dequeue_texture_buffer_method_  // (jominwoo): (Link [1])
                       : j_dequeue_byte_buffer_method_,
          dequeue_timeout_ms);

  if (CheckException(jni)) {
    ALOGE << "dequeueOutputBuffer() error";
    return false;
  }
  if (IsNull(jni, j_decoder_output_buffer)) {
    // No decoded frame ready.
    return true;
  }

  // Get decoded video frame properties.
  int color_format = GetIntField(jni, *j_media_codec_video_decoder_,
      j_color_format_field_);
  int width = GetIntField(jni, *j_media_codec_video_decoder_, j_width_field_);
  int height = GetIntField(jni, *j_media_codec_video_decoder_, j_height_field_);

  rtc::scoped_refptr<webrtc::VideoFrameBuffer> frame_buffer;
  int64_t presentation_timestamps_ms = 0;
  int64_t output_timestamps_ms = 0;
  int64_t output_ntp_timestamps_ms = 0;
  int decode_time_ms = 0;
  int64_t frame_delayed_ms = 0;
  if (use_surface_) {
    // Extract data from Java DecodedTextureBuffer.
    presentation_timestamps_ms = GetLongField(
        jni, j_decoder_output_buffer,
        j_texture_presentation_timestamp_ms_field_);
    output_timestamps_ms = GetLongField(
        jni, j_decoder_output_buffer, j_texture_timestamp_ms_field_);
    output_ntp_timestamps_ms = GetLongField(
        jni, j_decoder_output_buffer, j_texture_ntp_timestamp_ms_field_);
    decode_time_ms = GetLongField(
        jni, j_decoder_output_buffer, j_texture_decode_time_ms_field_);

    const int texture_id =
        GetIntField(jni, j_decoder_output_buffer, j_texture_id_field_);
    if (texture_id != 0) {  // |texture_id| == 0 represents a dropped frame.
      const jfloatArray j_transform_matrix =
          reinterpret_cast<jfloatArray>(GetObjectField(
              jni, j_decoder_output_buffer, j_transform_matrix_field_));
      frame_delayed_ms = GetLongField(
          jni, j_decoder_output_buffer, j_texture_frame_delay_ms_field_);

      // Create webrtc::VideoFrameBuffer with native texture handle.
      frame_buffer = surface_texture_helper_->CreateTextureFrame(
          width, height, NativeHandleImpl(jni, texture_id, j_transform_matrix));
    } else {
      EnableFrameLogOnWarning();
    }
  } else {
    // Extract data from Java ByteBuffer and create output yuv420 frame -
    // for non surface decoding only.
    int stride =
        GetIntField(jni, *j_media_codec_video_decoder_, j_stride_field_);
    const int slice_height =
        GetIntField(jni, *j_media_codec_video_decoder_, j_slice_height_field_);
    const int output_buffer_index = GetIntField(
        jni, j_decoder_output_buffer, j_info_index_field_);
    const int output_buffer_offset = GetIntField(
        jni, j_decoder_output_buffer, j_info_offset_field_);
    const int output_buffer_size = GetIntField(
        jni, j_decoder_output_buffer, j_info_size_field_);
    presentation_timestamps_ms = GetLongField(
        jni, j_decoder_output_buffer, j_presentation_timestamp_ms_field_);
    output_timestamps_ms = GetLongField(
        jni, j_decoder_output_buffer, j_timestamp_ms_field_);
    output_ntp_timestamps_ms = GetLongField(
        jni, j_decoder_output_buffer, j_ntp_timestamp_ms_field_);

    decode_time_ms = GetLongField(jni, j_decoder_output_buffer,
                                  j_byte_buffer_decode_time_ms_field_);
    RTC_CHECK_GE(slice_height, height);

    if (output_buffer_size < width * height * 3 / 2) {
      ALOGE << "Insufficient output buffer size: " << output_buffer_size;
      return false;
    }
    if (output_buffer_size < stride * height * 3 / 2 &&
        slice_height == height && stride > width) {
      // Some codecs (Exynos) incorrectly report stride information for
      // output byte buffer, so actual stride value need to be corrected.
      stride = output_buffer_size * 2 / (height * 3);
    }
    jobjectArray output_buffers = reinterpret_cast<jobjectArray>(GetObjectField(
        jni, *j_media_codec_video_decoder_, j_output_buffers_field_));
    jobject output_buffer =
        jni->GetObjectArrayElement(output_buffers, output_buffer_index);
    uint8_t* payload = reinterpret_cast<uint8_t*>(jni->GetDirectBufferAddress(
        output_buffer));
    if (CheckException(jni)) {
      return false;
    }
    payload += output_buffer_offset;

    // Create yuv420 frame.
    rtc::scoped_refptr<webrtc::I420Buffer> i420_buffer =
        decoded_frame_pool_.CreateBuffer(width, height);
    if (color_format == COLOR_FormatYUV420Planar) {
      RTC_CHECK_EQ(0, stride % 2);
      const int uv_stride = stride / 2;
      const uint8_t* y_ptr = payload;
      const uint8_t* u_ptr = y_ptr + stride * slice_height;

      // Note that the case with odd |slice_height| is handled in a special way.
      // The chroma height contained in the payload is rounded down instead of
      // up, making it one row less than what we expect in WebRTC. Therefore, we
      // have to duplicate the last chroma rows for this case. Also, the offset
      // between the Y plane and the U plane is unintuitive for this case. See
      // http://bugs.webrtc.org/6651 for more info.
      const int chroma_width = (width + 1) / 2;
      const int chroma_height =
          (slice_height % 2 == 0) ? (height + 1) / 2 : height / 2;
      const int u_offset = uv_stride * slice_height / 2;
      const uint8_t* v_ptr = u_ptr + u_offset;
      libyuv::CopyPlane(y_ptr, stride,
                        i420_buffer->MutableDataY(), i420_buffer->StrideY(),
                        width, height);
      libyuv::CopyPlane(u_ptr, uv_stride,
                        i420_buffer->MutableDataU(), i420_buffer->StrideU(),
                        chroma_width, chroma_height);
      libyuv::CopyPlane(v_ptr, uv_stride,
                        i420_buffer->MutableDataV(), i420_buffer->StrideV(),
                        chroma_width, chroma_height);
      if (slice_height % 2 == 1) {
        RTC_CHECK_EQ(height, slice_height);
        // Duplicate the last chroma rows.
        uint8_t* u_last_row_ptr = i420_buffer->MutableDataU() +
                                  chroma_height * i420_buffer->StrideU();
        memcpy(u_last_row_ptr, u_last_row_ptr - i420_buffer->StrideU(),
               i420_buffer->StrideU());
        uint8_t* v_last_row_ptr = i420_buffer->MutableDataV() +
                                  chroma_height * i420_buffer->StrideV();
        memcpy(v_last_row_ptr, v_last_row_ptr - i420_buffer->StrideV(),
               i420_buffer->StrideV());
      }
    } else {
      // All other supported formats are nv12.
      const uint8_t* y_ptr = payload;
      const uint8_t* uv_ptr = y_ptr + stride * slice_height;
      libyuv::NV12ToI420(y_ptr, stride, uv_ptr, stride,
                         i420_buffer->MutableDataY(), i420_buffer->StrideY(),
                         i420_buffer->MutableDataU(), i420_buffer->StrideU(),
                         i420_buffer->MutableDataV(), i420_buffer->StrideV(),
                         width, height);
    }
    frame_buffer = i420_buffer;

    // Return output byte buffer back to codec.
    jni->CallVoidMethod(
        *j_media_codec_video_decoder_,
        j_return_decoded_byte_buffer_method_,
        output_buffer_index);
    if (CheckException(jni)) {
      ALOGE << "returnDecodedOutputBuffer error";
      return false;
    }
  }
  if (frames_decoded_ < frames_decoded_logged_) {
    ALOGD << "Decoder frame out # " << frames_decoded_ <<
        ". " << width << " x " << height <<
        ". Color: " << color_format <<
        ". TS: " << presentation_timestamps_ms <<
        ". DecTime: " << (int)decode_time_ms <<
        ". DelayTime: " << (int)frame_delayed_ms;
  }

  // Calculate and print decoding statistics - every 3 seconds.
  frames_decoded_++;
  current_frames_++;
  current_decoding_time_ms_ += decode_time_ms;
  current_delay_time_ms_ += frame_delayed_ms;
  int statistic_time_ms = rtc::TimeMillis() - start_time_ms_;
  if (statistic_time_ms >= kMediaCodecStatisticsIntervalMs &&
      current_frames_ > 0) {
    int current_bitrate = current_bytes_ * 8 / statistic_time_ms;
    int current_fps =
        (current_frames_ * 1000 + statistic_time_ms / 2) / statistic_time_ms;
    ALOGD << "Frames decoded: " << frames_decoded_ <<
        ". Received: " <<  frames_received_ <<
        ". Bitrate: " << current_bitrate << " kbps" <<
        ". Fps: " << current_fps <<
        ". DecTime: " << (current_decoding_time_ms_ / current_frames_) <<
        ". DelayTime: " << (current_delay_time_ms_ / current_frames_) <<
        " for last " << statistic_time_ms << " ms.";
    start_time_ms_ = rtc::TimeMillis();
    current_frames_ = 0;
    current_bytes_ = 0;
    current_decoding_time_ms_ = 0;
    current_delay_time_ms_ = 0;
  }

  // If the frame was dropped, frame_buffer is left as nullptr.
  if (frame_buffer) {
    VideoFrame decoded_frame(frame_buffer, 0, 0, webrtc::kVideoRotation_0);
    decoded_frame.set_timestamp(output_timestamps_ms);
    decoded_frame.set_ntp_time_ms(output_ntp_timestamps_ms);

    rtc::Optional<uint8_t> qp = pending_frame_qps_.front();
    pending_frame_qps_.pop_front();
    callback_->Decoded(decoded_frame, rtc::Optional<int32_t>(decode_time_ms),
                       qp);
  }
  return true;
}
```

**Call Function Links**

| [1] | [MediaCodecVideoDecoder.dequeueTextureBuffer()](#mediacodecvideodecoderdequeuetexturebuffer) |
| [2] | [Class.Function()](#classfunction) |
| [3] | [Class.Function()](#classfunction) |
| [4] | [Class.Function()](#classfunction) |

<br>

### MediaCodecVideoDecoder.dequeueTextureBuffer()

**Function Information**

| File     | webrtc/sdk/android/api/org/webrtc/MediaCodecVideoDecoder.java |
| Class    | MediaCodecVideoDecoder |
| Function | dequeueTextureBuffer() |
| Description | |

**Source Code**

```java
  // Returns null if no decoded buffer is available, and otherwise a DecodedTextureBuffer.
  // Throws IllegalStateException if call is made on the wrong thread, if color format changes to an
  // unsupported format, or if |mediaCodec| is not in the Executing state. Throws CodecException
  // upon codec error. If |dequeueTimeoutMs| > 0, the oldest decoded frame will be dropped if
  // a frame can't be returned.
  private DecodedTextureBuffer dequeueTextureBuffer(int dequeueTimeoutMs) {
    checkOnMediaCodecThread();
    if (!useSurface) {
      throw new IllegalStateException("dequeueTexture() called for byte buffer decoding.");
    }
    DecodedOutputBuffer outputBuffer = dequeueOutputBuffer(dequeueTimeoutMs);
    if (outputBuffer != null) {
      dequeuedSurfaceOutputBuffers.add(outputBuffer);
    }

    MaybeRenderDecodedTextureBuffer();
    // Check if there is texture ready now by waiting max |dequeueTimeoutMs|.
    DecodedTextureBuffer renderedBuffer = textureListener.dequeueTextureBuffer(dequeueTimeoutMs);
    if (renderedBuffer != null) {
      MaybeRenderDecodedTextureBuffer();
      return renderedBuffer;
    }

    if ((dequeuedSurfaceOutputBuffers.size()
                >= Math.min(MAX_QUEUED_OUTPUTBUFFERS, outputBuffers.length)
            || (dequeueTimeoutMs > 0 && !dequeuedSurfaceOutputBuffers.isEmpty()))) {
      ++droppedFrames;
      // Drop the oldest frame still in dequeuedSurfaceOutputBuffers.
      // The oldest frame is owned by |textureListener| and can't be dropped since
      // mediaCodec.releaseOutputBuffer has already been called.
      final DecodedOutputBuffer droppedFrame = dequeuedSurfaceOutputBuffers.remove();
      if (dequeueTimeoutMs > 0) {
        // TODO(perkj): Re-add the below log when VideoRenderGUI has been removed or fixed to
        // return the one and only texture even if it does not render.
        Logging.w(TAG, "Draining decoder. Dropping frame with TS: "
                + droppedFrame.presentationTimeStampMs + ". Total number of dropped frames: "
                + droppedFrames);
      } else {
        Logging.w(TAG, "Too many output buffers " + dequeuedSurfaceOutputBuffers.size()
                + ". Dropping frame with TS: " + droppedFrame.presentationTimeStampMs
                + ". Total number of dropped frames: " + droppedFrames);
      }

      mediaCodec.releaseOutputBuffer(droppedFrame.index, false /* render */);
      return new DecodedTextureBuffer(0, null, droppedFrame.presentationTimeStampMs,
          droppedFrame.timeStampMs, droppedFrame.ntpTimeStampMs, droppedFrame.decodeTimeMs,
          SystemClock.elapsedRealtime() - droppedFrame.endDecodeTimeMs);
    }
    return null;
  }
```

**Call Function Links**

| [1] | [Class.Function()](#classfunction) |
| [2] | [Class.Function()](#classfunction) |
| [3] | [Class.Function()](#classfunction) |
| [4] | [Class.Function()](#classfunction) |

<br>

### Class.Function()

**Function Information**

| File     | location |
| Class    | Class |
| Function | Function() |
| Description | |

**Source Code**

```cpp
```

**Call Function Links**

| [1] | [Class.Function()](#classfunction) |
| [2] | [Class.Function()](#classfunction) |
| [3] | [Class.Function()](#classfunction) |
| [4] | [Class.Function()](#classfunction) |

<br>

### Class.Function()

**Function Information**

| File     | location |
| Class    | Class |
| Function | Function() |
| Description | |

**Source Code**

```cpp
```

**Call Function Links**

| [1] | [Class.Function()](#classfunction) |
| [2] | [Class.Function()](#classfunction) |
| [3] | [Class.Function()](#classfunction) |
| [4] | [Class.Function()](#classfunction) |

<br>

### Class.Function()

**Function Information**

| File     | location |
| Class    | Class |
| Function | Function() |
| Description | |

**Source Code**

```cpp
```

**Call Function Links**

| [1] | [Class.Function()](#classfunction) |
| [2] | [Class.Function()](#classfunction) |
| [3] | [Class.Function()](#classfunction) |
| [4] | [Class.Function()](#classfunction) |

<br>

### Class.Function()

**Function Information**

| File     | location |
| Class    | Class |
| Function | Function() |
| Description | |

**Source Code**

```cpp
```

**Call Function Links**

| [1] | [Class.Function()](#classfunction) |
| [2] | [Class.Function()](#classfunction) |
| [3] | [Class.Function()](#classfunction) |
| [4] | [Class.Function()](#classfunction) |

<br>

### Class.Function()

**Function Information**

| File     | location |
| Class    | Class |
| Function | Function() |
| Description | |

**Source Code**

```cpp
```

**Call Function Links**

| [1] | [Class.Function()](#classfunction) |
| [2] | [Class.Function()](#classfunction) |
| [3] | [Class.Function()](#classfunction) |
| [4] | [Class.Function()](#classfunction) |

<br>

### Class.Function()

**Function Information**

| File     | location |
| Class    | Class |
| Function | Function() |
| Description | |

**Source Code**

```cpp
```

**Call Function Links**

| [1] | [Class.Function()](#classfunction) |
| [2] | [Class.Function()](#classfunction) |
| [3] | [Class.Function()](#classfunction) |
| [4] | [Class.Function()](#classfunction) |

<br>
