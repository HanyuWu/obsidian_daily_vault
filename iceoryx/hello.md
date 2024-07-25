```c++
PoshRuntimeImpl(optional<const RuntimeName_t*> name,
                const DomainId domainId = DEFAULT_DOMAIN_ID,
                const RuntimeLocation location = RuntimeLocation::SEPARATE_PROCESS_FROM_ROUDI) noexcept;


PoshRuntimeImpl::PoshRuntimeImpl(optional<const RuntimeName_t*> name,
                                 const DomainId domainId,
                                 const RuntimeLocation location) noexcept
    : PoshRuntimeImpl(name, [&name, &domainId, &location] {
        auto runtimeInterfaceResult =
            IpcRuntimeInterface::create(*name.value(), domainId, runtime::PROCESS_WAITING_FOR_ROUDI_TIMEOUT);
        if (runtimeInterfaceResult.has_error())
        {
            switch (runtimeInterfaceResult.error())
            {
            case IpcRuntimeInterfaceError::CANNOT_CREATE_APPLICATION_CHANNEL:
                IOX_REPORT_FATAL(PoshError::IPC_INTERFACE__UNABLE_TO_CREATE_APPLICATION_CHANNEL);
            case IpcRuntimeInterfaceError::TIMEOUT_WAITING_FOR_ROUDI:
                IOX_LOG(FATAL, "Timeout registering at RouDi. Is RouDi running?");
                IOX_REPORT_FATAL(PoshError::IPC_INTERFACE__REG_ROUDI_NOT_AVAILABLE);
            case IpcRuntimeInterfaceError::SENDING_REQUEST_TO_ROUDI_FAILED:
                IOX_REPORT_FATAL(PoshError::IPC_INTERFACE__REG_UNABLE_TO_WRITE_TO_ROUDI_CHANNEL);
            case IpcRuntimeInterfaceError::NO_RESPONSE_FROM_ROUDI:
                IOX_REPORT_FATAL(PoshError::IPC_INTERFACE__REG_ACK_NO_RESPONSE);
            }

            IOX_UNREACHABLE();
        }
        auto& runtimeInterface = runtimeInterfaceResult.value();

        optional<SharedMemoryUser> shmInterface;

        // in case the runtime is located in the same process as RouDi the shm segments are already opened;
        // also in case of the RouDiEnv this would close the shm on destruction of the runtime which is also
        // not desired; therefore open the shm segments only when the runtime lives in a different process from RouDi
        if (location == RuntimeLocation::SEPARATE_PROCESS_FROM_ROUDI)
        {
            auto shmInterfaceResult = SharedMemoryUser::create(domainId,
                                                               runtimeInterface.getSegmentId(),
                                                               runtimeInterface.getShmTopicSize(),
                                                               runtimeInterface.getSegmentManagerAddressOffset());

            if (shmInterfaceResult.has_error())
            {
                switch (shmInterfaceResult.error())
                {
                case runtime::SharedMemoryUserError::SHM_MAPPING_ERROR:
                    IOX_REPORT_FATAL(PoshError::POSH__SHM_APP_MAPP_ERR);
                case runtime::SharedMemoryUserError::RELATIVE_POINTER_MAPPING_ERROR:
                    IOX_REPORT_FATAL(PoshError::POSH__SHM_APP_COULD_NOT_REGISTER_PTR_WITH_GIVEN_SEGMENT_ID);
                case runtime::SharedMemoryUserError::TOO_MANY_SHM_SEGMENTS:
                    IOX_REPORT_FATAL(PoshError::POSH__SHM_APP_SEGMENT_COUNT_OVERFLOW);
                }

                IOX_UNREACHABLE();
            }

            shmInterface.emplace(std::move(shmInterfaceResult.value()));
        }

        return std::pair<IpcRuntimeInterface, optional<SharedMemoryUser>>{std::move(runtimeInterface),
                                                                          std::move(shmInterface)};
    }())
{
    IOX_LOG(INFO, "Domain ID: " << static_cast<DomainId::value_type>(domainId));
}

```