import { usePost } from "@bfirst/api-client";
import { StoryForm } from "./components/StoryForm";

interface FeatureStoryCreateProps {
  onSuccess?: () => void;
  onError: (error) => void;
}

export const FeatureStoryCreate: React.FC<FeatureStoryCreateProps> = (props: FeatureStoryCreateProps) => {
  const { request, isError, isPending, isSuccess, error } = usePost("api/v1/stories");

  if (isError) {
    props.onError && props.onError(error);
  }

  if (isSuccess) {
    props.onSuccess && props.onSuccess();
  }

  return (
    <StoryForm
      btnLabel="Publish"
      isError={false}
      loading={isPending}
      onSubmit={(data) => {
        request(data);
      }}
    />
  );
};
