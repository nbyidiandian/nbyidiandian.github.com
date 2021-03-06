---
layout: post
title: "dynamic protocol buffer message"
description: ""
category: protobuf
tags: [c++, protobuf, serialize, reflection]
---
{% include JB/setup %}

关于protobuf的self-describing message，可以看[google的文档](https://developers.google.com/protocol-buffers/docs/techniques?hl=zh-CN#self-description) <br />

贴上我的测试代码：<br />

    #include <iostream>
    #include <google/protobuf/descriptor.h>
    #include <google/protobuf/descriptor.pb.h>
    #include <google/protobuf/dynamic_message.h>
    
    using namespace std;
    using namespace google::protobuf;
    
    int main(int argc, const char *argv[])
    {
        FileDescriptorProto file_proto;
        file_proto.set_name("foo.proto");
    
        // create dynamic message proto names "Pair"
        DescriptorProto *message_proto = file_proto.add_message_type();
        message_proto->set_name("Pair");
    
        FieldDescriptorProto *field_proto = NULL;
    
        field_proto = message_proto->add_field();
        field_proto->set_name("key");
        field_proto->set_type(FieldDescriptorProto::TYPE_STRING);
        field_proto->set_number(1);
        field_proto->set_label(FieldDescriptorProto::LABEL_REQUIRED);
    
        field_proto = message_proto->add_field();
        field_proto->set_name("value");
        field_proto->set_type(FieldDescriptorProto::TYPE_UINT32);
        field_proto->set_number(2);
        field_proto->set_label(FieldDescriptorProto::LABEL_REQUIRED);
    
        // add the "Pair" message proto to file proto and build it
        DescriptorPool pool;
        const FileDescriptor *file_descriptor = pool.BuildFile(file_proto);
        const Descriptor *descriptor = file_descriptor->FindMessageTypeByName("Pair");
    
        cout << descriptor->DebugString();
    
        // build a dynamic message by "Pair" proto
        DynamicMessageFactory factory(&pool);
        const Message *message = factory.GetPrototype(descriptor);
    
        // create a real instance of "Pair"
        Message *pair = message->New();
    
        // write the "Pair" instance by reflection
        const Reflection *reflection = pair->GetReflection();
    
        const FieldDescriptor *field = NULL;
        field = descriptor->FindFieldByName("key");
        reflection->SetString(pair, field, "my key");
        field = descriptor->FindFieldByName("value");
        reflection->SetUInt32(pair, field, 1234);
    
        cout << pair->DebugString();
    
        delete pair;
    
        return 0;
    }


编译、链接，运行下看：<br />

    message Pair {
      required string key = 1;
      required uint32 value = 2;
    }
    key: "my key"
    value: 1234

是不是有点意思了。<br />
关于dynamic message的序列化/反序列化、以及dynamic message如何与static proto message一起使用，还待进一步学习。<br />
未完待续...
